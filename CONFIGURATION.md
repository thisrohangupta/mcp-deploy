# Configuration Reference

This document provides detailed configuration options for the Harness.io MCP Server deployment.

## Table of Contents

- [Environment Variables](#environment-variables)
- [Toolsets](#toolsets)
- [Resource Configuration](#resource-configuration)
- [Health Checks](#health-checks)
- [Scaling](#scaling)
- [Self-Hosted Harness](#self-hosted-harness)
- [Examples](#examples)

## Environment Variables

### Required Variables

| Variable | Source | Description |
|----------|--------|-------------|
| `HARNESS_API_KEY` | Secret | API key for authenticating with Harness.io. Generate at Account Settings > API Keys. |
| `HARNESS_DEFAULT_ORG_ID` | ConfigMap | Your Harness organization identifier. Found in the URL: `app.harness.io/ng/account/{accountId}/home/orgs/{orgId}` |
| `HARNESS_DEFAULT_PROJECT_ID` | ConfigMap | Your Harness project identifier. Found in project settings or URL. |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HARNESS_BASE_URL` | `https://app.harness.io` | Base URL for Harness API. Change for self-hosted installations. |
| `HARNESS_TOOLSETS` | `default` | Comma-separated list of enabled toolsets. See [Toolsets](#toolsets). |
| `HARNESS_LOG_LEVEL` | `info` | Logging verbosity. Options: `debug`, `info`, `warn`, `error`. |
| `HARNESS_ACCOUNT_ID` | - | Explicitly set account ID (usually auto-detected from API key). |

## Toolsets

Toolsets define which Harness capabilities are exposed through the MCP server.

### Available Toolsets

| Toolset | Description | Use Cases |
|---------|-------------|-----------|
| `default` | Core MCP functionality including authentication and basic operations | Always include this |
| `pipelines` | Pipeline CRUD, execution, and monitoring | CI/CD automation, deployment triggers |
| `repositories` | Code repository management and operations | Source control integration |
| `connectors` | Connector configuration and management | Integration setup, credential management |

### Configuring Toolsets

In `deploy/configmap.yaml`:

```yaml
data:
  # Enable all toolsets
  HARNESS_TOOLSETS: "default,pipelines,repositories,connectors"

  # Minimal - only core functionality
  HARNESS_TOOLSETS: "default"

  # CI/CD focused
  HARNESS_TOOLSETS: "default,pipelines"
```

### Security Consideration

Only enable toolsets that are required for your use case. Each toolset exposes additional API surface area.

## Resource Configuration

### Default Resources

The deployment includes resource requests and limits in `deploy/deployment.yaml`:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Resource Tuning Guidelines

| Workload | Memory Request | Memory Limit | CPU Request | CPU Limit |
|----------|----------------|--------------|-------------|-----------|
| Light (< 10 req/min) | 128Mi | 256Mi | 50m | 200m |
| Medium (10-100 req/min) | 128Mi | 512Mi | 100m | 500m |
| Heavy (> 100 req/min) | 256Mi | 1Gi | 200m | 1000m |

### Adjusting Resources

Edit `deploy/deployment.yaml`:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

## Health Checks

### Liveness Probe

Determines if the container should be restarted:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10  # Wait before first check
  periodSeconds: 30        # Check every 30 seconds
  timeoutSeconds: 1        # Timeout for each check
  failureThreshold: 3      # Restart after 3 failures
```

### Readiness Probe

Determines if the pod should receive traffic:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5   # Wait before first check
  periodSeconds: 10        # Check every 10 seconds
  timeoutSeconds: 1        # Timeout for each check
  failureThreshold: 3      # Remove from service after 3 failures
```

### Tuning Health Checks

For slower startup environments:

```yaml
livenessProbe:
  initialDelaySeconds: 30  # Increase for slow starts
  periodSeconds: 60        # Less frequent checks
readinessProbe:
  initialDelaySeconds: 10
  periodSeconds: 15
```

## Scaling

### Replica Count

Default deployment runs 2 replicas for high availability:

```yaml
spec:
  replicas: 2
```

### Horizontal Pod Autoscaler (Optional)

Create an HPA for automatic scaling:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: harness-mcp-server-hpa
  namespace: harness-mcp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: harness-mcp-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

Save as `deploy/hpa.yaml` and apply:

```bash
kubectl apply -f deploy/hpa.yaml
```

## Self-Hosted Harness

For self-hosted Harness installations, update the base URL:

### On-Premises Installation

```yaml
# deploy/configmap.yaml
data:
  HARNESS_BASE_URL: "https://harness.your-company.com"
```

### Custom Port

```yaml
data:
  HARNESS_BASE_URL: "https://harness.your-company.com:8443"
```

### With Custom CA Certificate

If your self-hosted installation uses a custom CA:

1. Create a ConfigMap with your CA certificate:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harness-ca-cert
  namespace: harness-mcp
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    YOUR_CA_CERTIFICATE_HERE
    -----END CERTIFICATE-----
```

2. Mount it in the deployment:

```yaml
spec:
  containers:
    - name: mcp-server
      volumeMounts:
        - name: ca-cert
          mountPath: /etc/ssl/certs/harness-ca.crt
          subPath: ca.crt
  volumes:
    - name: ca-cert
      configMap:
        name: harness-ca-cert
```

## Examples

### Minimal Configuration

For a basic deployment with core functionality:

**configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harness-mcp-config
  namespace: harness-mcp
data:
  HARNESS_DEFAULT_ORG_ID: "default"
  HARNESS_DEFAULT_PROJECT_ID: "my-project"
  HARNESS_TOOLSETS: "default"
  HARNESS_LOG_LEVEL: "info"
```

### Full-Featured Configuration

For comprehensive Harness integration:

**configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harness-mcp-config
  namespace: harness-mcp
data:
  HARNESS_BASE_URL: "https://app.harness.io"
  HARNESS_DEFAULT_ORG_ID: "engineering"
  HARNESS_DEFAULT_PROJECT_ID: "platform"
  HARNESS_TOOLSETS: "default,pipelines,repositories,connectors"
  HARNESS_LOG_LEVEL: "debug"
```

### Development/Debug Configuration

For troubleshooting:

**configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harness-mcp-config
  namespace: harness-mcp
data:
  HARNESS_BASE_URL: "https://app.harness.io"
  HARNESS_DEFAULT_ORG_ID: "dev-org"
  HARNESS_DEFAULT_PROJECT_ID: "test-project"
  HARNESS_TOOLSETS: "default,pipelines"
  HARNESS_LOG_LEVEL: "debug"  # Verbose logging
```

**deployment.yaml adjustments:**
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "256Mi"
    cpu: "250m"
replicas: 1  # Single replica for dev
```

## Applying Configuration Changes

After modifying ConfigMap values, restart the deployment to apply:

```bash
# Apply updated ConfigMap
kubectl apply -f deploy/configmap.yaml

# Restart pods to pick up changes
kubectl rollout restart deployment/harness-mcp-server -n harness-mcp

# Watch rollout progress
kubectl rollout status deployment/harness-mcp-server -n harness-mcp
```

## Validating Configuration

Check that environment variables are correctly set:

```bash
# Get a pod name
POD=$(kubectl get pods -n harness-mcp -l app=harness-mcp-server -o jsonpath='{.items[0].metadata.name}')

# Check environment variables
kubectl exec -n harness-mcp $POD -- env | grep HARNESS

# Check logs for configuration issues
kubectl logs -n harness-mcp $POD | head -50
```

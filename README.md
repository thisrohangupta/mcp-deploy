# Harness.io MCP Server - Kubernetes Deployment

Deploy the [Harness.io MCP Server](https://github.com/harness/harness-mcp-server) on Kubernetes to enable AI models to interact with your Harness.io platform via the Model Context Protocol (MCP).

## Overview

This repository provides production-ready Kubernetes manifests to deploy the Harness.io MCP server with:

- High availability (2 replicas)
- Security hardening (non-root, read-only filesystem)
- Health checks and resource management
- Configurable toolsets for pipelines, repositories, and connectors

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                 Namespace: harness-mcp                 │  │
│  │                                                        │  │
│  │  ┌──────────────┐    ┌──────────────┐                 │  │
│  │  │   Pod (1)    │    │   Pod (2)    │   Deployment    │  │
│  │  │  mcp-server  │    │  mcp-server  │   (2 replicas)  │  │
│  │  └──────┬───────┘    └──────┬───────┘                 │  │
│  │         │                   │                          │  │
│  │         └─────────┬─────────┘                          │  │
│  │                   │                                    │  │
│  │         ┌─────────▼─────────┐                          │  │
│  │         │  Service :8080    │  ClusterIP               │  │
│  │         └─────────┬─────────┘                          │  │
│  │                   │                                    │  │
│  │    ┌──────────────┴──────────────┐                     │  │
│  │    │                             │                     │  │
│  │    ▼                             ▼                     │  │
│  │ ConfigMap                     Secret                   │  │
│  │ (harness-mcp-config)    (harness-mcp-secrets)          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Harness.io API │
                    │ app.harness.io  │
                    └─────────────────┘
```

## Prerequisites

- Kubernetes cluster (v1.21+)
- `kubectl` configured with cluster access
- Harness.io account with API access
- Harness API key ([Generate one here](https://developer.harness.io/docs/platform/automation/api/add-and-manage-api-keys/))

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/thisrohangupta/mcp-deploy.git
cd mcp-deploy
```

### 2. Configure Your Credentials

Edit the configuration files with your Harness.io details:

**ConfigMap** (`deploy/configmap.yaml`):
```yaml
data:
  HARNESS_BASE_URL: "https://app.harness.io"
  HARNESS_DEFAULT_ORG_ID: "your-org-id"          # Replace with your org ID
  HARNESS_DEFAULT_PROJECT_ID: "your-project-id"  # Replace with your project ID
  HARNESS_TOOLSETS: "default,pipelines,repositories,connectors"
  HARNESS_LOG_LEVEL: "info"
```

**Secret** (`deploy/namespace.yaml`):
```yaml
stringData:
  HARNESS_API_KEY: "your-api-key"  # Replace with your Harness API key
```

### 3. Deploy to Kubernetes

```bash
# Apply all manifests in order
kubectl apply -f deploy/namespace.yaml
kubectl apply -f deploy/configmap.yaml
kubectl apply -f deploy/deployment.yaml
kubectl apply -f deploy/service.yaml
```

Or apply all at once:
```bash
kubectl apply -f deploy/
```

### 4. Verify Deployment

```bash
# Check pod status
kubectl get pods -n harness-mcp

# Check service
kubectl get svc -n harness-mcp

# View logs
kubectl logs -n harness-mcp -l app=harness-mcp-server -f
```

### 5. Test the Server

```bash
# Port-forward to test locally
kubectl port-forward -n harness-mcp svc/harness-mcp-server 8080:8080

# Health check
curl http://localhost:8080/health
```

## Configuration Reference

See [CONFIGURATION.md](./CONFIGURATION.md) for detailed configuration options.

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `HARNESS_API_KEY` | Yes | - | Your Harness.io API key (stored in Secret) |
| `HARNESS_BASE_URL` | No | `https://app.harness.io` | Harness.io API base URL |
| `HARNESS_DEFAULT_ORG_ID` | Yes | - | Default organization identifier |
| `HARNESS_DEFAULT_PROJECT_ID` | Yes | - | Default project identifier |
| `HARNESS_TOOLSETS` | No | `default` | Comma-separated list of enabled toolsets |
| `HARNESS_LOG_LEVEL` | No | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |

### Available Toolsets

| Toolset | Description |
|---------|-------------|
| `default` | Core MCP functionality |
| `pipelines` | CI/CD pipeline management |
| `repositories` | Code repository operations |
| `connectors` | Integration connectors management |

## Security Best Practices

See [SECURITY.md](./SECURITY.md) for comprehensive security guidance.

### Current Security Features

- **Non-root execution**: Container runs as non-root user
- **Read-only filesystem**: Prevents runtime modifications
- **No privilege escalation**: `allowPrivilegeEscalation: false`
- **Resource limits**: Prevents resource exhaustion
- **Secrets management**: API key stored in Kubernetes Secret

### Recommendations for Production

1. **Use External Secrets**: Integrate with HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault
2. **Add NetworkPolicy**: Restrict pod-to-pod communication
3. **Pin image versions**: Avoid using `latest` tag
4. **Enable audit logging**: Track API access
5. **Use RBAC**: Create dedicated ServiceAccount with minimal permissions

## Troubleshooting

### Pods not starting

```bash
# Check pod events
kubectl describe pod -n harness-mcp -l app=harness-mcp-server

# Check logs
kubectl logs -n harness-mcp -l app=harness-mcp-server --previous
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `CrashLoopBackOff` | Invalid API key or configuration | Verify `HARNESS_API_KEY` and org/project IDs |
| `ImagePullBackOff` | Cannot pull container image | Check network access to Docker Hub |
| Health check failing | Service not ready | Increase `initialDelaySeconds` in probes |
| Connection refused | Service not exposed | Verify Service is created and port-forward is active |

### Validating Configuration

```bash
# Test API key validity
curl -H "x-api-key: YOUR_API_KEY" \
  https://app.harness.io/gateway/ng/api/user/currentUser

# Check ConfigMap
kubectl get configmap harness-mcp-config -n harness-mcp -o yaml

# Check Secret (base64 encoded)
kubectl get secret harness-mcp-secrets -n harness-mcp -o yaml
```

## Updating the Deployment

### Rolling Update

```bash
# Update image version
kubectl set image deployment/harness-mcp-server \
  mcp-server=harness/mcp-server:v1.2.0 \
  -n harness-mcp

# Watch rollout
kubectl rollout status deployment/harness-mcp-server -n harness-mcp
```

### Configuration Changes

```bash
# Edit ConfigMap
kubectl edit configmap harness-mcp-config -n harness-mcp

# Restart pods to pick up changes
kubectl rollout restart deployment/harness-mcp-server -n harness-mcp
```

## Cleanup

```bash
# Delete all resources
kubectl delete -f deploy/

# Or delete namespace (removes everything)
kubectl delete namespace harness-mcp
```

## File Structure

```
mcp-deploy/
├── README.md              # This file
├── CONFIGURATION.md       # Detailed configuration reference
├── SECURITY.md            # Security best practices
└── deploy/
    ├── namespace.yaml     # Namespace and Secret
    ├── configmap.yaml     # Configuration
    ├── deployment.yaml    # Deployment spec
    └── service.yaml       # Service definition
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Resources

- [Harness.io MCP Server](https://github.com/harness/harness-mcp-server)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Harness.io Documentation](https://developer.harness.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

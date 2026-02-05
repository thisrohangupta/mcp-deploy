# Security Best Practices

This document outlines security considerations and best practices for deploying the Harness.io MCP Server in production environments.

## Table of Contents

- [Current Security Posture](#current-security-posture)
- [API Key Management](#api-key-management)
- [Network Security](#network-security)
- [Container Security](#container-security)
- [RBAC Configuration](#rbac-configuration)
- [Secrets Management](#secrets-management)
- [Monitoring and Auditing](#monitoring-and-auditing)
- [Production Checklist](#production-checklist)

## Current Security Posture

The default deployment includes these security measures:

| Feature | Status | Description |
|---------|--------|-------------|
| Non-root execution | Enabled | `runAsNonRoot: true` |
| Read-only filesystem | Enabled | `readOnlyRootFilesystem: true` |
| Privilege escalation | Disabled | `allowPrivilegeEscalation: false` |
| Resource limits | Configured | Memory and CPU limits set |
| Secrets in Secret object | Enabled | API key stored as Kubernetes Secret |

## API Key Management

### Principle of Least Privilege

Create a dedicated API key with minimal required permissions:

1. Go to Harness.io > Account Settings > Access Control
2. Create a new Service Account for MCP
3. Assign only required roles:
   - **Viewer** for read-only operations
   - **Pipeline Executor** if triggering pipelines
   - **Project Admin** only if managing resources

### API Key Rotation

Implement regular key rotation:

```bash
# 1. Generate new API key in Harness UI

# 2. Update Kubernetes Secret
kubectl create secret generic harness-mcp-secrets \
  --from-literal=HARNESS_API_KEY='new-api-key' \
  --namespace harness-mcp \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Restart deployment to pick up new key
kubectl rollout restart deployment/harness-mcp-server -n harness-mcp

# 4. Revoke old API key in Harness UI after confirming new key works
```

### Recommended Key Settings

When creating API keys:
- Set expiration (30-90 days recommended)
- Use descriptive names (e.g., `mcp-server-production`)
- Enable IP restrictions if possible
- Document key purpose and owner

## Network Security

### NetworkPolicy

Restrict ingress and egress traffic. Create `deploy/networkpolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: harness-mcp-server-netpol
  namespace: harness-mcp
spec:
  podSelector:
    matchLabels:
      app: harness-mcp-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow traffic only from specific namespaces/pods
    - from:
        - namespaceSelector:
            matchLabels:
              name: allowed-namespace
        - podSelector:
            matchLabels:
              app: allowed-client
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Allow HTTPS to Harness.io
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

Apply:
```bash
kubectl apply -f deploy/networkpolicy.yaml
```

### Service Exposure

The default Service type is `ClusterIP` (internal only). For external access:

**Option 1: Ingress with TLS (Recommended)**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: harness-mcp-server-ingress
  namespace: harness-mcp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - mcp.your-domain.com
      secretName: mcp-tls-secret
  rules:
    - host: mcp.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: harness-mcp-server
                port:
                  number: 8080
```

**Option 2: Internal Load Balancer**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: harness-mcp-server-internal
  namespace: harness-mcp
  annotations:
    # For AWS
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    # For GCP
    cloud.google.com/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: harness-mcp-server
  ports:
    - port: 443
      targetPort: 8080
```

## Container Security

### Pod Security Standards

Apply Pod Security Standards at namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: harness-mcp
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Image Security

**Pin Image Versions**

Avoid `latest` tag. Use specific versions or digests:

```yaml
# Version tag
image: harness/mcp-server:v1.2.0

# Or SHA digest (most secure)
image: harness/mcp-server@sha256:abc123...
```

**Image Pull Policy**

```yaml
imagePullPolicy: Always  # For version tags
# or
imagePullPolicy: IfNotPresent  # For digest-based references
```

**Private Registry**

If using a private registry:

```yaml
spec:
  imagePullSecrets:
    - name: registry-credentials
  containers:
    - name: mcp-server
      image: private.registry.com/harness/mcp-server:v1.2.0
```

### Security Context (Enhanced)

Full security context configuration:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: mcp-server
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

## RBAC Configuration

### Dedicated ServiceAccount

Create a restricted ServiceAccount:

```yaml
# deploy/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: harness-mcp-server
  namespace: harness-mcp
automountServiceAccountToken: false  # Disable if not needed
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: harness-mcp-server-role
  namespace: harness-mcp
rules:
  # Only add rules if the application needs Kubernetes API access
  # Empty rules = no Kubernetes API access
  []
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: harness-mcp-server-binding
  namespace: harness-mcp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: harness-mcp-server-role
subjects:
  - kind: ServiceAccount
    name: harness-mcp-server
    namespace: harness-mcp
```

Update the Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: harness-mcp-server
      automountServiceAccountToken: false
```

## Secrets Management

### External Secrets Operator

Integrate with external secret stores:

**AWS Secrets Manager Example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harness-mcp-external-secret
  namespace: harness-mcp
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: harness-mcp-secrets
    creationPolicy: Owner
  data:
    - secretKey: HARNESS_API_KEY
      remoteRef:
        key: harness/mcp-server/api-key
```

**HashiCorp Vault Example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harness-mcp-vault-secret
  namespace: harness-mcp
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: harness-mcp-secrets
  data:
    - secretKey: HARNESS_API_KEY
      remoteRef:
        key: secret/data/harness/mcp
        property: api_key
```

### Sealed Secrets

For GitOps workflows, use Bitnami Sealed Secrets:

```bash
# Encrypt the secret
kubeseal --format=yaml < secret.yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to git (safe to store)
```

## Monitoring and Auditing

### Prometheus Metrics

Add annotations for Prometheus scraping:

```yaml
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
```

### Audit Logging

Enable Kubernetes audit logging for the namespace:

```yaml
# In your audit policy
- level: RequestResponse
  namespaces: ["harness-mcp"]
  verbs: ["create", "update", "patch", "delete"]
  resources:
    - group: ""
      resources: ["secrets", "configmaps"]
```

### Log Aggregation

Ensure logs are forwarded to your SIEM/logging platform:

```bash
# View logs for security analysis
kubectl logs -n harness-mcp -l app=harness-mcp-server --since=24h
```

## Production Checklist

Before deploying to production, verify:

### Secrets
- [ ] API key has minimal required permissions
- [ ] API key has expiration set
- [ ] Secrets stored in external secret manager (not in YAML files)
- [ ] Secret rotation process documented

### Network
- [ ] NetworkPolicy applied
- [ ] Service not exposed publicly (or secured with auth)
- [ ] TLS enabled for all external communication
- [ ] Egress restricted to required destinations

### Container
- [ ] Image version pinned (not `latest`)
- [ ] Image from trusted registry
- [ ] Pod Security Standards enforced
- [ ] Resource limits appropriate for workload

### Access Control
- [ ] Dedicated ServiceAccount created
- [ ] RBAC configured with least privilege
- [ ] Namespace isolation verified

### Operations
- [ ] Monitoring and alerting configured
- [ ] Log aggregation enabled
- [ ] Backup/recovery procedures documented
- [ ] Incident response plan in place

### Compliance
- [ ] Data handling requirements reviewed
- [ ] Retention policies configured
- [ ] Access audit trail enabled

## Reporting Security Issues

If you discover a security vulnerability:

1. **Do not** open a public GitHub issue
2. Contact the maintainers privately
3. Provide detailed reproduction steps
4. Allow time for a fix before disclosure

## Additional Resources

- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [External Secrets Operator](https://external-secrets.io/)
- [Harness Security Documentation](https://developer.harness.io/docs/platform/security/)

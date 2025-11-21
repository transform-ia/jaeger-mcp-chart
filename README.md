# Jaeger MCP Chart

Helm chart for deploying jaeger-mcp server to Kubernetes. Provides MCP (Model Context
Protocol) access to Jaeger distributed tracing data.

## Overview

This chart deploys the jaeger-mcp server which exposes Jaeger tracing data through the
MCP protocol, enabling AI assistants and other tools to query and analyze distributed traces.

**Key Features:**

- MCP server exposing Jaeger Query API
- Security-hardened deployment (read-only filesystem, non-root user)
- Network policies for controlled access
- Configurable Jaeger connection settings
- Resource limits and requests

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Existing Jaeger deployment with Query service

## Installation

### Quick Start

```bash
# Install from GitHub Container Registry
helm install jaeger-mcp oci://ghcr.io/transform-ia/jaeger-mcp-chart \
  --namespace claude-tools \
  --create-namespace
```

### Install from Source

```bash
# Clone the repository
git clone https://github.com/transform-ia/jaeger-mcp-chart.git
cd jaeger-mcp-chart

# Install the chart
helm install jaeger-mcp . \
  --namespace claude-tools \
  --create-namespace
```

### Custom Configuration

```bash
# Create custom values file
cat > custom-values.yaml <<EOF
jaeger:
  url: "http://my-jaeger.monitoring.svc.cluster.local"
  port: "16686"

resources:
  limits:
    cpu: 500m
    memory: 256Mi
EOF

# Install with custom values
helm install jaeger-mcp . -f custom-values.yaml
```

## Accessing Jaeger Web UI

The jaeger-mcp server connects to an existing Jaeger deployment. To access the Jaeger Web UI:

### Method 1: Port Forward (Recommended for Development)

```bash
# Port forward to Jaeger Query service
kubectl port-forward -n claude-tools svc/jaeger-query 16686:16686

# Open browser to http://localhost:16686
```

### Method 2: Ingress (Recommended for Production)

Create an Ingress resource for the Jaeger Query service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger-ui
  namespace: claude-tools
spec:
  rules:
    - host: jaeger.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jaeger-query
                port:
                  number: 16686
```

Apply and access at `http://jaeger.example.com`

### Method 3: NodePort Service

```bash
# Patch the Jaeger Query service to NodePort
kubectl patch svc jaeger-query -n claude-tools -p '{"spec":{"type":"NodePort"}}'

# Get the NodePort
kubectl get svc jaeger-query -n claude-tools -o jsonpath='{.spec.ports[?(@.name=="query")].nodePort}'

# Access at http://<node-ip>:<nodeport>
```

## Configuration

### Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `transform-ia/jaeger-mcp` |
| `image.tag` | Image tag | `""` (uses appVersion) |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `81` |
| `service.targetPort` | Container port | `8000` |
| `jaeger.url` | Jaeger Query URL | `http://jaeger-query.claude-tools.svc.cluster.local` |
| `jaeger.port` | Jaeger Query port | `80` |
| `jaeger.protocol` | Protocol (HTTP/GRPC) | `HTTP` |
| `env.logLevel` | Log level | `info` |
| `resources.limits.cpu` | CPU limit | `200m` |
| `resources.limits.memory` | Memory limit | `128Mi` |
| `resources.requests.cpu` | CPU request | `50m` |
| `resources.requests.memory` | Memory request | `64Mi` |
| `networkPolicies.enabled` | Enable network policies | `true` |

### Example Configurations

#### Connect to Different Jaeger Instance

```yaml
jaeger:
  url: "http://jaeger-query.monitoring.svc.cluster.local"
  port: "16686"
  protocol: "HTTP"
```

#### Increase Resources

```yaml
resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

#### Disable Network Policies

```yaml
networkPolicies:
  enabled: false
```

## Usage

### MCP Endpoint

After deployment, the MCP server is accessible at:

```text
http://<release-name>-jaeger-mcp-chart.<namespace>.svc.cluster.local:81/mcp
```

Example with default release name and namespace:

```text
http://jaeger-mcp-jaeger-mcp-chart.claude-tools.svc.cluster.local:81/mcp
```

### Verify Installation

```bash
# Check pod status
kubectl get pods -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart

# Check service
kubectl get svc -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart

# Check logs
kubectl logs -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart

# Test MCP endpoint (from within cluster)
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://jaeger-mcp-jaeger-mcp-chart.claude-tools.svc.cluster.local:81/mcp
```

### Integration with Claude Code

Add the MCP server to Claude Code's configuration:

```json
{
  "mcpServers": {
    "jaeger": {
      "type": "http",
      "url": "http://jaeger-mcp-jaeger-mcp-chart.claude-tools.svc.cluster.local:81/mcp"
    }
  }
}
```

## Architecture

### Components

- **Deployment**: Single replica pod running jaeger-mcp server
- **Service**: ClusterIP service exposing MCP endpoint on port 81
- **NetworkPolicy**: Controls ingress from Claude namespace and egress to Jaeger

### Security

The deployment includes security hardening:

- **Read-only root filesystem**: Prevents runtime modifications
- **Non-root user**: Runs as UID 1000
- **Dropped capabilities**: Minimal Linux capabilities
- **Network policies**: Restricted ingress/egress
- **Resource limits**: CPU and memory constraints

### Network Policies

By default, the chart creates network policies that:

1. **Allow ingress** from pods in the `claude` namespace with label `app=claude-code`
2. **Allow egress** to pods in the `claude-tools` namespace (Jaeger Query)
3. **Allow egress** to kube-dns for DNS resolution

## Upgrading

```bash
# Upgrade to latest version
helm upgrade jaeger-mcp oci://ghcr.io/transform-ia/jaeger-mcp-chart \
  --namespace claude-tools

# Upgrade with custom values
helm upgrade jaeger-mcp . -f custom-values.yaml
```

## Uninstallation

```bash
# Uninstall the chart
helm uninstall jaeger-mcp -n claude-tools

# Optionally delete the namespace
kubectl delete namespace claude-tools
```

## Troubleshooting

### Pod Not Starting

```bash
# Describe the pod
kubectl describe pod -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart

# Check events
kubectl get events -n claude-tools --sort-by='.lastTimestamp'
```

### Connection Issues

```bash
# Test connection to Jaeger Query service
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n claude-tools -- \
  curl -v http://jaeger-query.claude-tools.svc.cluster.local/api/services

# Check network policies
kubectl get networkpolicies -n claude-tools
```

### View Logs

```bash
# View logs
kubectl logs -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart --tail=100 -f

# View previous logs (if pod restarted)
kubectl logs -n claude-tools -l app.kubernetes.io/name=jaeger-mcp-chart --previous
```

## Development

### Local Testing

```bash
# Lint the chart
helm lint .

# Render templates
helm template test . --debug

# Dry run installation
helm install test . --dry-run --debug
```

### CI/CD

The chart uses GitHub Actions for automated CI/CD:

1. **Lint**: YAML and Markdown linting on every push/PR
2. **Package & Push**: Automatic packaging and publishing to GHCR on tag push

To release a new version:

```bash
git tag v0.0.2
git push origin v0.0.2
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally with `helm lint` and `helm template`
5. Submit a pull request

## License

MIT

## Maintainers

- Transform IA (<bruno.clermont@gmail.com>)

## Related Projects

- **Jaeger MCP Server**: <https://github.com/transform-ia/jaeger-mcp-server>
- **Jaeger**: <https://www.jaegertracing.io/>
- **Model Context Protocol**: <https://modelcontextprotocol.io/>

## Support

For issues and questions:

- GitHub Issues: <https://github.com/transform-ia/jaeger-mcp-chart/issues>
- Jaeger Documentation: <https://www.jaegertracing.io/docs/>

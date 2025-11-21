# CLAUDE.md

This file provides guidance to Claude Code when working with the jaeger-mcp-chart Helm chart.

## Overview

This Helm chart deploys the jaeger-mcp server to Kubernetes, providing MCP (Model Context
Protocol) access to Jaeger distributed tracing data. The chart includes:

- Deployment with security-hardened container
- ClusterIP Service for MCP endpoint
- Network policies for controlled access
- Configurable Jaeger connection settings

## Chart Structure

```text
jaeger-mcp-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── templates/              # Kubernetes resource templates
│   ├── _helpers.tpl        # Template helpers
│   ├── deployment.yaml     # Pod deployment
│   ├── service.yaml        # ClusterIP service
│   └── networkpolicy.yaml  # Network policies
├── .github/                # GitHub Actions workflows
│   └── workflows/
│       └── ci.yaml         # CI/CD pipeline
├── .helmignore             # Files to ignore during packaging
├── .yamllint.yaml          # YAML linting configuration
├── .markdownlint.yaml      # Markdown linting configuration
├── CLAUDE.md               # This file
└── README.md               # User documentation
```

## Development Workflow

### Local Testing

```bash
# Lint the chart
helm lint .

# Render templates
helm template test . --debug

# Dry run installation
helm install test . --dry-run --debug
```

### Publishing

The chart is automatically published to GitHub Container Registry when a tag is pushed:

```bash
# Tag a new version
git tag v0.0.1
git push origin v0.0.1
```

GitHub Actions will:

1. Lint YAML and Markdown files
2. Update Chart.yaml version to match the tag
3. Package the Helm chart
4. Push to `oci://ghcr.io/transform-ia/jaeger-mcp-chart`

### Deployment

**Use the k8s-manager agent** to deploy this chart via ArgoCD:

```markdown
User: "Deploy jaeger-mcp using the new chart"
→ Launch k8s-manager agent
→ Agent creates ArgoCD Application pointing to oci://ghcr.io/transform-ia/jaeger-mcp-chart
→ Agent commits to /workspace/applications/
→ root-app auto-syncs the application
```

## Configuration

### Key Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | jaeger-mcp image repository | `transform-ia/jaeger-mcp` |
| `image.tag` | Image tag | `""` (uses appVersion) |
| `service.port` | MCP server port | `81` |
| `jaeger.url` | Jaeger Query service URL | `http://jaeger-query.claude-tools.svc.cluster.local` |
| `jaeger.port` | Jaeger Query port | `80` |
| `jaeger.protocol` | Protocol (HTTP/GRPC) | `HTTP` |
| `networkPolicies.enabled` | Enable network policies | `true` |

### Example Custom Values

```yaml
# custom-values.yaml
jaeger:
  url: "http://my-jaeger.monitoring.svc.cluster.local"
  port: "16686"

resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

## Integration with Jaeger

This chart connects to an existing Jaeger deployment. The default configuration expects:

- Jaeger Query service at `jaeger-query.claude-tools.svc.cluster.local:80`
- HTTP protocol for queries

To access the Jaeger Web UI:

```bash
# Port forward to Jaeger Query service
kubectl port-forward -n claude-tools svc/jaeger-query 16686:16686

# Open browser to http://localhost:16686
```

## Security

The deployment includes:

- Read-only root filesystem
- Non-root user (UID 1000)
- Dropped capabilities
- Network policies for ingress/egress control

## MCP Endpoint

After deployment, the MCP server is available at:

```text
http://<release-name>-jaeger-mcp-chart.<namespace>.svc.cluster.local:81/mcp
```

Example with release name `jaeger-mcp` in namespace `claude-tools`:

```text
http://jaeger-mcp-jaeger-mcp-chart.claude-tools.svc.cluster.local:81/mcp
```

## Making Changes

When modifying the chart:

1. Update templates in `templates/` directory
2. Update `values.yaml` if adding new configuration options
3. Test locally with `helm lint` and `helm template`
4. Update README.md with any new features or configuration
5. Commit changes
6. Tag new version: `git tag v0.0.x`
7. Push: `git push && git push --tags`
8. Wait for GitHub Actions to pass
9. Update ArgoCD Application to use new version

## Related Resources

- **Docker Image**: `transform-ia/jaeger-mcp` (Docker Hub)
- **Source Code**: `https://github.com/transform-ia/jaeger-mcp-server`
- **Chart Repository**: `https://github.com/transform-ia/jaeger-mcp-chart`
- **Published Chart**: `oci://ghcr.io/transform-ia/jaeger-mcp-chart`

## Best Practices

1. **Version Everything**: Always tag releases with semantic versioning
2. **Test Before Publishing**: Run helm lint and helm template before pushing tags
3. **Document Changes**: Update README.md with new configuration options
4. **Security First**: Maintain security context and network policies
5. **Use ArgoCD**: Deploy via ArgoCD for GitOps compliance

## Troubleshooting

### Chart not packaging

- Check YAML syntax with `helm lint .`
- Verify Chart.yaml has required fields
- Check .helmignore isn't excluding necessary files

### Linting failures

- Run `yamllint -c .yamllint.yaml .` locally
- Run `markdownlint-cli2 '**/*.md'` for markdown
- Fix reported issues before pushing

### GitHub Actions failing

- Check workflow logs in GitHub Actions tab
- Verify Chart.yaml syntax
- Ensure version tag format is `vX.Y.Z`

## Further Reading

- **Helm Documentation**: <https://helm.sh/docs/>
- **k8s-manager Agent**: `/workspace/.claude/agents/k8s-manager.md`
- **ArgoCD Documentation**: `/workspace/deployments/argocd/README.md`

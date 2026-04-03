# GoClaw Helm Charts

Helm charts for deploying [GoClaw](https://github.com/nextlevelbuilder/goclaw) — a multi-agent AI gateway.

## Charts

| Chart | Description |
|-------|-------------|
| [goclaw](charts/goclaw/) | GoClaw server, UI, optional PostgreSQL, and optional GCPlane GitOps control plane |

## Quick Start

```bash
# From source
git clone https://github.com/dataplanelabs/goclaw-charts.git
helm install goclaw ./goclaw-charts/charts/goclaw -f your-values.yaml
```

## Configuration

See [charts/goclaw/values.yaml](charts/goclaw/values.yaml) for all available options.

### Components

| Component | Default | Description |
|-----------|---------|-------------|
| Server | enabled | GoClaw API server (port 18790) |
| UI | enabled | Web dashboard (port 80) |
| PostgreSQL | enabled | Sidecar database with pgvector |
| Chrome | enabled | Browser sidecar for web automation |
| GCPlane | disabled | GitOps control plane for declarative resource management |

### Image Registries

GoClaw images are published to multiple registries. Override `image.repository` to use a different source:

| Registry | Server Image | Web UI Image |
|----------|-------------|--------------|
| GHCR (default) | `ghcr.io/nextlevelbuilder/goclaw` | `ghcr.io/nextlevelbuilder/goclaw-web` |
| Docker Hub | `digitop/goclaw` | `digitop/goclaw-web` |

**Image tag variants** (append to version, e.g., `v2.65.0-full`):

| Variant | Suffix | Description |
|---------|--------|-------------|
| Standard | _(none)_ | Backend + embedded UI + Python |
| Base | `-base` | Lightweight API-only, no UI or runtimes |
| Full | `-full` | All runtimes + skill dependencies pre-installed |
| OTel | `-otel` | Standard + OpenTelemetry instrumentation |

**Private registry example:**

```yaml
global:
  imagePullSecrets:
    - name: my-registry-credentials

image:
  repository: my-registry.example.com/goclaw
  tag: "v2.65.0-full"

ui:
  image:
    repository: my-registry.example.com/goclaw-web
    tag: "v2.65.0"
```

### Database Options

| Option | Use Case | Config |
|--------|----------|--------|
| Sidecar PostgreSQL | Dev/local | `db.enabled: true` (default) |
| External PostgreSQL | Production | `db.enabled: false` + `externalDatabase.url: postgres://...` |

The sidecar uses [pgvector/pgvector:pg18](https://hub.docker.com/r/pgvector/pgvector) — PostgreSQL 18 with the pgvector extension for vector search.

For production, disable the sidecar and point to a managed PostgreSQL:

```yaml
db:
  enabled: false

externalDatabase:
  url: "postgres://user:password@host:5432/goclaw?sslmode=require"
  # Or use individual fields:
  # host: "my-rds-instance.amazonaws.com"
  # port: 5432
  # name: "goclaw"
  # user: "goclaw"
  # existingSecret: "my-db-secret"  # must contain POSTGRES_PASSWORD key
```

### GCPlane (GitOps Auto-Deployment)

[GCPlane](https://github.com/dataplanelabs/gcplane) is a declarative GitOps control plane that continuously reconciles GoClaw resources (providers, agents, channels, MCP servers, etc.) from YAML manifests.

Enable GCPlane with a custom manifest:

```yaml
gcplane:
  enabled: true
  interval: "30s"
  webhook:
    url: "https://hooks.slack.com/services/..."
    format: "slack"
  manifest: |
    apiVersion: gcplane/v1
    kind: Manifest
    metadata:
      name: my-goclaw
    spec:
      provider:
        url: http://goclaw:18790
    resources:
      - kind: Provider
        name: anthropic
        spec:
          type: anthropic
          apiKey: ${ANTHROPIC_API_KEY}
      - kind: Agent
        name: assistant
        spec:
          name: Assistant
          model: claude-sonnet-4-20250514
          provider: anthropic
```

GCPlane automatically uses the GoClaw gateway token from the chart's secret. Use `${GOCLAW_TOKEN}` in manifests for token substitution.

Pass additional secrets (API keys) via `gcplane.extraEnv`:

```yaml
gcplane:
  extraEnv:
    - name: ANTHROPIC_API_KEY
      valueFrom:
        secretKeyRef:
          name: my-api-keys
          key: anthropic
```

### All Default Images

| Component | Default Image |
|-----------|---------------|
| Server | `ghcr.io/nextlevelbuilder/goclaw:v2.65.0-full` |
| UI | `ghcr.io/nextlevelbuilder/goclaw-web:v2.65.0` |
| GCPlane | `ghcr.io/dataplanelabs/gcplane:latest` |
| Database | `pgvector/pgvector:pg18` |
| Chrome | `zenika/alpine-chrome:124` |

## License

Apache 2.0

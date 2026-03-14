# GoClaw Helm Charts

Helm charts for deploying [GoClaw](https://github.com/nextlevelbuilder/goclaw) — a multi-agent AI gateway.

## Charts

| Chart | Description |
|-------|-------------|
| [goclaw](charts/goclaw/) | Main chart: server, UI, and optional sidecar PostgreSQL |

## Quick Start

```bash
# From source
git clone https://github.com/dataplanelabs/goclaw-charts.git
helm install goclaw ./goclaw-charts/charts/goclaw -f your-values.yaml
```

## Configuration

See [charts/goclaw/values.yaml](charts/goclaw/values.yaml) for all available options.

### Database Options

| Option | Use Case | Config |
|--------|----------|--------|
| Sidecar PostgreSQL | Dev/local | `db.enabled: true` |
| External PostgreSQL | Production | `externalDatabase.url: postgres://...` |

## License

Apache 2.0

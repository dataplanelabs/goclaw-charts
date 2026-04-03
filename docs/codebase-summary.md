# GoClaw Charts - Codebase Summary

## Directory Structure

```
goclaw-charts/
├── Chart.yaml                  # Helm chart metadata (v0.2.0, appVersion v2.65.0)
├── values.yaml                 # Complete values configuration with inline documentation
├── charts/
│   └── goclaw/
│       ├── Chart.yaml          # Chart definition
│       ├── values.yaml         # Default values and config
│       └── templates/
│           ├── _helpers.tpl              # 9 reusable template helper functions
│           ├── NOTES.txt                 # Post-install instructions
│           ├── configmap.yaml            # Server environment variables
│           ├── configmap-gcplane.yaml    # GCPlane manifest ConfigMap
│           ├── deployment.yaml           # GoClaw server + Chrome sidecar
│           ├── deployment-ui.yaml        # Web UI deployment
│           ├── deployment-db.yaml        # PostgreSQL sidecar deployment
│           ├── deployment-gcplane.yaml   # GCPlane GitOps control plane
│           ├── ingress.yaml              # Ingress with path-based routing
│           ├── pdb.yaml                  # PodDisruptionBudgets
│           ├── pvc.yaml                  # PersistentVolumeClaims
│           ├── secret.yaml               # Auto-generated secrets
│           ├── service.yaml              # Server ClusterIP service
│           ├── service-ui.yaml           # UI ClusterIP service
│           ├── service-db.yaml           # Database ClusterIP service (internal)
│           └── service-gcplane.yaml      # GCPlane ClusterIP service
└── docs/                       # Documentation (this directory)
```

## Template Files Overview

### _helpers.tpl
**Purpose**: Reusable template functions for consistency and DRY principle.

**Helper Functions** (9 total):
| Function | Purpose | Output |
|----------|---------|--------|
| `goclaw.name` | Chart name (truncated to 63 chars) | `goclaw` |
| `goclaw.fullname` | Full release name (release or fullnameOverride) | `my-release` |
| `goclaw.labels` | Standard Kubernetes labels (app, version, managed-by) | Label key=value pairs |
| `goclaw.selectorLabels` | Pod selector labels (app.kubernetes.io/name, instance) | Selector key=value pairs |
| `goclaw.dbHost` | Database hostname (sidecar or external) | `my-release-db` or external host |
| `goclaw.dbPort` | Database port (5432 for sidecar, from values for external) | `5432` or custom port |
| `goclaw.dbName` | Database name (`goclaw` for sidecar, from values for external) | `goclaw` or custom name |
| `goclaw.dbUser` | Database user (`goclaw` for sidecar, from values for external) | `goclaw` or custom user |
| `goclaw.secretName` | Secret name (existingSecret or auto-generated) | Secret resource name |

**Usage Pattern**:
```yaml
{{ include "goclaw.fullname" . }}        # Renders release name
{{ include "goclaw.labels" . | nindent 4 }} # Renders indented label block
```

---

### configmap.yaml
**Purpose**: Store environment variables for GoClaw server (non-secret configuration).

**Key Environment Variables**:
| Key | Value | Source |
|-----|-------|--------|
| `GOCLAW_HOST` | `0.0.0.0` | Fixed (listen on all interfaces) |
| `GOCLAW_PORT` | `{{ .Values.server.port }}` | Default: 18790 |
| `GOCLAW_DATA_DIR` | `/app/data` | Fixed (data PVC mount point) |
| `GOCLAW_WORKSPACE_DIR` | `/app/workspace` | Fixed (workspace PVC mount point) |
| `GOCLAW_BROWSER_URL` | `ws://localhost:{{ .Values.browser.port }}` | Default: 9222 (Chrome sidecar) |
| `GOCLAW_CONFIG` | `/app/data/config.json` | Only if `persistence.data.enabled: true` |

**Conditional Rendering**:
- `GOCLAW_BROWSER_URL` omitted if `browser.enabled: false`
- `GOCLAW_CONFIG` set only when data PVC is enabled
- `GOCLAW_WORKSPACE_DIR` omitted if `persistence.workspace.enabled: false`

---

### configmap-gcplane.yaml
**Purpose**: Store GCPlane declarative manifest for resource auto-deployment.

**Features**:
- Conditionally created only if `gcplane.enabled: true`
- Supports user-provided manifest or auto-generated default
- Variable substitution: `${GOCLAW_TOKEN}` replaced at runtime with actual token value
- ConfigMap checksum included in GCPlane deployment annotations to trigger pod restarts on manifest changes

**Auto-Generated Default Manifest**:
```yaml
apiVersion: gcplane/v1
kind: Manifest
metadata:
  name: goclaw-default
spec:
  provider:
    url: http://{{ .Values.server.fullname }}-server:18790
  resources: []
```

---

### deployment.yaml
**Purpose**: Deploy GoClaw server with Chrome browser sidecar and database wait init container.

**Containers**:
| Container | Purpose | Image | Mount Points |
|-----------|---------|-------|--------------|
| `server` | Main GoClaw API | `ghcr.io/nextlevelbuilder/goclaw:v2.65.0-full` | `/app/data`, `/app/workspace`, `/tmp` |
| `browser` | Chrome DevTools Protocol sidecar | `zenika/alpine-chrome:124` | `/dev/shm` (shared memory) |

**Init Container**:
- `wait-for-db`: Uses `busybox` to wait for PostgreSQL TCP port (5432) readiness (only if `db.enabled: true`)

**Security Context**:
- `runAsUser: 1000` (non-root)
- `runAsNonRoot: true`
- `readOnlyRootFilesystem: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`

**Environment Configuration**:
- ConfigMap mounted as env source (`envFrom.configMapRef`)
- Secret mounted as env source (`envFrom.secretRef`)
- `POSTGRES_PASSWORD` injected from secret for DSN construction
- `GOCLAW_POSTGRES_DSN` built dynamically from host/port/user/password/name
- Optional `GOCLAW_CONFIG` set if persistence enabled

**Probes**:
- **Startup**: Path `/health`, 5s initial delay, 3s period, 10 failures threshold (30s total)
- **Readiness**: Path `/health`, 10s period, 3s timeout
- **Liveness**: Path `/health`, 30s period, 3s timeout

**Volume Mounts**:
- `/tmp` (emptyDir)
- `/app/data` (data PVC, if enabled)
- `/app/workspace` (workspace PVC, if enabled)
- `/dev/shm` (Chrome sidecar, 2Gi shared memory)

**Deployment Strategy**:
- **RollingUpdate** (default): 1 surge pod, 0 unavailable during update (safe for RWO PVCs on same node)
- **Recreate**: Alternative for multi-node clusters with RWO PVCs (avoid mount conflicts)

---

### deployment-ui.yaml
**Purpose**: Deploy web UI (nginx-based, read-only asset serving).

**Container**:
| Container | Image | Port |
|-----------|-------|------|
| `ui` | `ghcr.io/nextlevelbuilder/goclaw-web:v2.65.0` | 80 |

**Security Context**:
- `runAsUser: 101` (nginx standard non-root user)
- `readOnlyRootFilesystem: false` (nginx needs /var/cache, /var/run writable)
- `allowPrivilegeEscalation: false`

**Volume Mounts**:
- `/tmp`, `/var/cache/nginx`, `/var/run` (emptyDir for nginx runtime)

**Probes**: Identical to server (startup, readiness, liveness)

---

### deployment-db.yaml
**Purpose**: Deploy PostgreSQL sidecar with pgvector extension.

**Container**:
| Container | Image | Port |
|-----------|-------|------|
| `db` | `pgvector/pgvector:pg18` | 5432 |

**Security Context**:
- `runAsUser: 999` (PostgreSQL system user)
- `readOnlyRootFilesystem: true`

**Environment**:
- `POSTGRES_USER: goclaw`
- `POSTGRES_PASSWORD` from secret
- `POSTGRES_DB: goclaw`

**Volume Mounts**:
- `/var/lib/postgresql/data` (db PVC)
- `/tmp` (emptyDir)

**Strategy**: Recreate (not RollingUpdate, for data consistency)

**Probes**:
- **Startup**: pg_isready check, 10s initial delay, 3s period, 10 failures threshold
- **Readiness**: pg_isready check, 10s period, 3s timeout
- **Liveness**: pg_isready check, 30s period, 3s timeout

---

### deployment-gcplane.yaml
**Purpose**: Deploy GCPlane GitOps control plane for declarative resource reconciliation.

**Container**:
| Container | Image | Port |
|-----------|-------|------|
| `gcplane` | `ghcr.io/dataplanelabs/gcplane:latest` | 8480 |

**Environment**:
- `GOCLAW_TOKEN`: Injected from chart secret
- `GOCLAW_URL`: `http://{{ .Values.server.fullname }}:18790`
- `MANIFEST_PATH`: `/config/manifest.yaml`
- `RECONCILE_INTERVAL`: `{{ .Values.gcplane.interval }}` (default: 30s)
- `LOG_FORMAT`: `{{ .Values.gcplane.logFormat }}` (default: json)
- `WEBHOOK_URL`: Optional drift notification webhook
- `WEBHOOK_FORMAT`: Slack/Discord/Teams/Google Chat/Telegram

**Volume Mounts**:
- `/config` (manifest ConfigMap)

**Pod Annotation**:
- `gcplane.dataplanelabs.io/manifest-checksum`: Detects manifest changes in ConfigMap; restarts pod on update

---

### ingress.yaml
**Purpose**: Path-based HTTP(S) routing to server, UI, and health endpoints.

**Conditions**: Created only if `ingress.enabled: true`

**Routes**:
| Path | Backend | Service | Port |
|------|---------|---------|------|
| `/ws` | Server WebSocket | `{{ fullname }}` | 18790 |
| `/v1` | Server REST API | `{{ fullname }}` | 18790 |
| `/health` | Server health | `{{ fullname }}` | 18790 |
| `/` | UI web dashboard | `{{ fullname }}-ui` | 80 |

**TLS**:
- Enabled if `ingress.tls: true`
- Certificate issuer configured via annotations (e.g., cert-manager)

**Annotations**:
- User-provided via `ingress.annotations`
- Common: `cert-manager.io/cluster-issuer`, `nginx.ingress.kubernetes.io/rewrite-target`

---

### pdb.yaml
**Purpose**: Pod Disruption Budget to prevent voluntary evictions during cluster maintenance.

**Conditions**: Created for server and UI if `podDisruptionBudget.enabled: true`

**Configuration**:
- `minAvailable` or `maxUnavailable` (user chooses one via values)
- Default: disabled (recommended for single-replica dev deployments)

---

### pvc.yaml
**Purpose**: Allocate persistent storage for data, workspace, and database.

**PVCs**:
| PVC | Size | Path | Purpose | StorageClass |
|-----|------|------|---------|--------------|
| `data` | `persistence.data.size` (5Gi) | `/app/data` | GoClaw config/state | `persistence.data.storageClass` |
| `workspace` | `persistence.workspace.size` (10Gi) | `/app/workspace` | Agent artifacts | `persistence.workspace.storageClass` |
| `db` | `db.storage` (20Gi) | `/var/lib/postgresql/data` | PostgreSQL data | `db.storageClass` |

**Access Mode**: ReadWriteOnce (RWO)

**Conditional Creation**:
- Data/workspace PVCs created only if `persistence.{data,workspace}.enabled: true`
- DB PVC created only if `db.enabled: true`

---

### secret.yaml
**Purpose**: Store sensitive configuration (passwords, tokens, encryption keys).

**Auto-Generated Secrets** (only if `config.existingSecret` empty):
| Key | Generation | Value |
|-----|-----------|-------|
| `GOCLAW_GATEWAY_TOKEN` | `randAlphaNum 32` | API authentication token |
| `GOCLAW_ENCRYPTION_KEY` | `randAlphaNum 32` | Data encryption key |
| `POSTGRES_PASSWORD` | `randAlphaNum 32` | Database password |

**User-Provided Mode**:
- If `config.existingSecret` specified, skip creation and reference external secret
- If `externalDatabase.existingSecret` specified, use for DB password instead

**Protection**:
- `helm.sh/resource-policy: keep` prevents accidental deletion during `helm uninstall`

---

### service.yaml, service-ui.yaml, service-db.yaml
**Purpose**: Internal and external network exposure for deployments.

| Service | Type | Port | Target | Internal |
|---------|------|------|--------|----------|
| `{{ fullname }}` | ClusterIP | 18790 | server:18790 | Yes |
| `{{ fullname }}-ui` | ClusterIP | 80 | ui:80 | No (web-facing) |
| `{{ fullname }}-db` | ClusterIP | 5432 | db:5432 | Yes (internal only) |
| `{{ fullname }}-gcplane` | ClusterIP | 8480 | gcplane:8480 | Yes |

**Prometheus Annotations** (GCPlane service):
- `prometheus.io/scrape: "true"`
- `prometheus.io/port: "8480"`
- Enables auto-discovery for Prometheus monitoring

---

### NOTES.txt
**Purpose**: Post-install user instructions and troubleshooting.

**Content**:
- Deployment confirmation with version
- Access instructions (port-forward or ingress URL)
- GCPlane status check commands (if enabled)
- Manifest reconciliation interval and webhook info

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `Chart.yaml` | 14 | Chart metadata |
| `values.yaml` | 199 | Configuration with 30+ documented parameters |
| `_helpers.tpl` | 92 | 9 helper functions |
| `deployment.yaml` | 120+ | Server + Chrome + DB wait + probes + security |
| `deployment-ui.yaml` | 80+ | Web UI with emptyDir volumes |
| `deployment-db.yaml` | 70+ | PostgreSQL with pgvector |
| `ingress.yaml` | 45+ | Path-based routing |
| `secret.yaml` | 30+ | Auto-generated + protection policy |
| `pvc.yaml` | 60+ | 3 PVCs with conditional creation |

---

## Template Variables Reference

### Global/Shared
```
.Chart.Name              = "goclaw"
.Chart.Version           = "0.2.0"
.Chart.AppVersion        = "v2.65.0"
.Release.Name            = user-provided release name
.Release.Namespace       = current namespace
.Capabilities.APIVersions = Kubernetes API versions
```

### Values Usage (Common)
```
.Values.global.imagePullSecrets     # Private registry credentials
.Values.image.repository            # Server image repo
.Values.image.tag                   # Server image tag
.Values.server.replicas             # Number of server pods
.Values.server.port                 # Server port (18790)
.Values.server.strategy.type        # RollingUpdate or Recreate
.Values.server.resources.{requests,limits}  # CPU/memory
.Values.server.podDisruptionBudget  # PDB config
.Values.ui.enabled                  # Deploy UI? (true/false)
.Values.ui.replicas                 # Number of UI pods
.Values.db.enabled                  # Deploy sidecar DB? (true/false)
.Values.db.image                    # PostgreSQL image
.Values.externalDatabase.host       # External DB hostname
.Values.persistence.{data,workspace}.enabled  # PVC creation
.Values.persistence.{data,workspace}.size     # PVC size
.Values.ingress.enabled             # Deploy ingress? (true/false)
.Values.ingress.host                # Ingress hostname
.Values.config.gatewayToken         # Pre-set gateway token
.Values.config.existingSecret       # Reference external secret
.Values.gcplane.enabled             # Deploy GCPlane? (true/false)
.Values.gcplane.manifest            # User-provided manifest YAML
.Values.browser.enabled             # Deploy Chrome sidecar? (true/false)
```

---

## Helm Functions Used

| Function | Purpose | Example |
|----------|---------|---------|
| `include` | Include helper template | `{{ include "goclaw.fullname" . }}` |
| `toYaml` | Marshal data to YAML | `{{ .Values.server.resources \| toYaml }}` |
| `nindent` | Indent multi-line YAML | `{{ toYaml ... \| nindent 8 }}` |
| `printf` | Format strings | `{{ printf "%s-%s" .Chart.Name .Chart.Version }}` |
| `default` | Provide default value | `{{ .Values.image.tag \| default .Chart.AppVersion }}` |
| `if`/`else` | Conditional rendering | `{{ if .Values.db.enabled }} ... {{ end }}` |
| `with` | Change context | `{{ with .Values.server.resources }} ... {{ end }}` |
| `range` | Iterate lists | `{{ range .Values.server.extraEnv }} ... {{ end }}` |
| `quote` | Add quotes | `{{ .Values.server.port \| quote }}` |
| `eq`, `ne`, `lt`, `gt` | Comparisons | `{{ if eq .Values.server.replicas 1 }} ... {{ end }}` |

---

## Configuration Patterns

### Conditional Component Deployment
```yaml
# values.yaml
server.enabled: true    # Always on
ui.enabled: true        # Toggle
db.enabled: true        # Toggle
browser.enabled: true   # Toggle
gcplane.enabled: false  # Toggle
```

```yaml
# template
{{ if .Values.ui.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "goclaw.fullname" . }}-ui
...
{{ end }}
```

### Dynamic Database Configuration
```yaml
# Template detects sidecar vs external
{{ if .Values.db.enabled }}
  # Sidecar mode: construct DSN from fixed values
  GOCLAW_POSTGRES_DSN: postgres://goclaw:password@{{ fullname }}-db:5432/goclaw?sslmode=disable
{{ else if .Values.externalDatabase.url }}
  # External mode: user-provided DSN
  GOCLAW_POSTGRES_DSN: {{ .Values.externalDatabase.url }}
{{ else }}
  # External mode: construct from parameters
  GOCLAW_POSTGRES_DSN: postgres://{{ .Values.externalDatabase.user }}:password@{{ .Values.externalDatabase.host }}:{{ .Values.externalDatabase.port }}/{{ .Values.externalDatabase.name }}?sslmode=disable
{{ end }}
```

### Secret Reference Pattern
```yaml
# Use existing secret or auto-generated
secretKeyRef:
  name: {{ include "goclaw.secretName" . }}
  key: POSTGRES_PASSWORD
```

---

## Security Hardening Features

1. **Non-Root Execution**: All containers run with numeric UID (1000, 999, 101)
2. **Read-Only Root**: All containers except UI have `readOnlyRootFilesystem: true`
3. **No Privilege Escalation**: `allowPrivilegeEscalation: false` on all containers
4. **Dropped Capabilities**: `capabilities.drop: ["ALL"]` enforced
5. **Secret Protection**: `helm.sh/resource-policy: keep` prevents secret deletion
6. **Init Container Gating**: DB wait init container blocks server start; no auto-retry
7. **Image Pull Secrets**: Support private registries via `global.imagePullSecrets`

---

## Common Customization Points

| Use Case | Values to Modify |
|----------|-----------------|
| **Multi-Replica HA** | `server.replicas: 3`, `server.strategy.type: RollingUpdate`, `db.enabled: false`, `externalDatabase.host: ...` |
| **Production Scaling** | `server.resources.limits`, `ui.resources.limits`, `browser.resources.limits`, `persistence.*.size` |
| **Custom Domain** | `ingress.enabled: true`, `ingress.host: api.example.com`, `ingress.tls: true` |
| **Private Registry** | `global.imagePullSecrets: [{name: ghcr-secret}]`, update image repos |
| **GCPlane GitOps** | `gcplane.enabled: true`, `gcplane.manifest: <yaml>`, `gcplane.webhook.url: <webhook>` |
| **Custom Image Variant** | `image.tag: v2.65.0-otel` (for OpenTelemetry), `image.tag: v2.65.0-base` (API-only) |
| **Existing Namespace Secrets** | `config.existingSecret: my-secret`, `externalDatabase.existingSecret: db-secret` |


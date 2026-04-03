# GoClaw Charts - System Architecture

## Component Architecture Diagram

```mermaid
graph TB
    User["👤 User/Client"]
    IGW["Internet Gateway<br/>(Ingress)"]
    
    subgraph K8s["Kubernetes Cluster"]
        subgraph Namespace["Namespace: goclaw"]
            subgraph ServerPods["Server Deployment<br/>(1+ Pods)"]
                Server1["Server Pod"]
                Chrome1["Chrome Sidecar<br/>Port 9222"]
                Server1 -.->|CDP| Chrome1
            end
            
            subgraph UIPods["UI Deployment<br/>(Optional, 1+ Pods)"]
                UI1["Web UI Pod<br/>Port 80"]
            end
            
            subgraph DBPod["Database Deployment<br/>(Optional, 1 Pod)"]
                DB["PostgreSQL<br/>+ pgvector<br/>Port 5432"]
            end
            
            subgraph GCPlanePods["GCPlane Deployment<br/>(Optional, 1+ Pods)"]
                GCPlane["GCPlane Pod<br/>Port 8480"]
                Manifest["Manifest ConfigMap"]
                GCPlane -.->|reads| Manifest
            end
            
            subgraph Storage["Persistent Storage"]
                DataPVC["Data PVC<br/>/app/data<br/>5Gi"]
                WorkspacePVC["Workspace PVC<br/>/app/workspace<br/>10Gi"]
                DBPVC["DB PVC<br/>/var/lib/postgresql/data<br/>20Gi"]
            end
            
            subgraph Secrets["Secrets & ConfigMaps"]
                Secret["Secret<br/>GATEWAY_TOKEN<br/>ENCRYPTION_KEY<br/>POSTGRES_PASSWORD"]
                Config["ConfigMap<br/>GOCLAW_HOST<br/>GOCLAW_PORT<br/>GOCLAW_BROWSER_URL"]
            end
            
            subgraph Services["Services"]
                ServerSvc["Service: goclaw<br/>ClusterIP:18790"]
                UISVC["Service: goclaw-ui<br/>ClusterIP:80"]
                DBSVC["Service: goclaw-db<br/>ClusterIP:5432<br/>(internal)"]
                GCPlaneSvc["Service: goclaw-gcplane<br/>ClusterIP:8480"]
            end
            
            Server1 -->|writes| DataPVC
            Server1 -->|writes| WorkspacePVC
            Server1 -->|reads| Secret
            Server1 -->|reads| Config
            Server1 -->|connects| DB
            Server1 -->|authenticates| DBSVC
            
            UI1 -->|reads| Config
            
            DB -->|reads/writes| DBPVC
            DB -->|reads| Secret
            
            GCPlane -->|reads| Secret
            GCPlane -->|reads| Config
            GCPlane -->|HTTP API| ServerSvc
            
            ServerSvc <--> Server1
            UISVC <--> UI1
            DBSVC <--> DB
            GCPlaneSvc <--> GCPlane
        end
    end
    
    User -->|HTTPS /ws, /v1, /health| IGW
    User -->|HTTPS /| IGW
    IGW -->|HTTP| ServerSvc
    IGW -->|HTTP| UISVC
    
    ExtDB["External PostgreSQL<br/>(Prod, Optional)"]
    ServerSvc -.->|DSN| ExtDB
    
    style K8s fill:#e1f5ff
    style Namespace fill:#fff3e0
    style ServerPods fill:#f3e5f5
    style UIPods fill:#e8f5e9
    style DBPod fill:#fce4ec
    style GCPlanePods fill:#f1f8e9
    style Storage fill:#ede7f6
    style Secrets fill:#fff9c4
    style Services fill:#c8e6c9
```

---

## Data Flow

### Request Flow: User to GoClaw Server

```
1. User (external or in-cluster) → 2. Ingress Controller
   ├─ Route: /ws, /v1, /health → GoClaw Server Service
   └─ Route: / → GoClaw UI Service

3. Kubernetes Service (ClusterIP:18790)
   ├─ Load-balances across Server pods
   └─ Resolves to Pod IP via kube-dns

4. Server Pod
   ├─ Receives request on port 18790
   ├─ Queries PostgreSQL (via Chrome sidecar or direct call)
   ├─ Stores/retrieves data from /app/data PVC
   └─ Returns response
```

### Database Flow: Server ↔ PostgreSQL

```
Sidecar Mode (db.enabled: true):
1. Server pod waits for DB pod readiness (init container: wait-for-db)
2. Server env POSTGRES_PASSWORD injected from secret
3. Server constructs DSN: postgres://goclaw:password@goclaw-db:5432/goclaw?sslmode=disable
4. Server connects via Kubernetes Service discovery (DNS)
5. Data persisted to DB PVC (ReadWriteOnce, single pod)

External Mode (db.enabled: false):
1. User provides externalDatabase.host, port, user, name, or complete DSN
2. Server uses provided DSN directly (no password injection via env)
3. No init container wait required
4. Multi-replica servers can safely connect to external DB
```

### GCPlane Flow: Manifest Reconciliation

```
1. GCPlane pod reads manifest from ConfigMap (/config/manifest.yaml)
2. Parses manifest YAML with ${GOCLAW_TOKEN} variable substitution
3. Extracts provider URL and resources (providers, agents, channels, etc.)
4. Connects to GoClaw Server via ClusterIP:18790 with injected GOCLAW_TOKEN
5. Reconciles resources: create/update/delete providers and agents
6. On drift (actual ≠ desired): sends webhook notification (Slack, Discord, etc.)
7. Repeats every {{ .Values.gcplane.interval }} (default: 30s)
```

---

## Component Details

### Server Component

**Role**: Main GoClaw API gateway

**Responsibilities**:
- Accept client requests (REST, WebSocket)
- Route requests to configured AI providers
- Manage agent state and execution
- Persist data to PostgreSQL
- Provide health/readiness endpoints

**Ports**:
- `18790` (HTTP) — API and WebSocket connections

**Dependencies**:
- PostgreSQL database (sidecar or external)
- Chrome sidecar (optional) for web automation

**Resource Defaults**:
- Request: 100m CPU, 256Mi RAM
- Limit: 2000m CPU, 1Gi RAM

**Probes**:
- Startup: `/health` every 3s, 10 failures (30s total grace period)
- Readiness: `/health` every 10s
- Liveness: `/health` every 30s

**Security**:
- UID 1000 (non-root)
- Read-only root filesystem
- No privilege escalation
- All capabilities dropped

---

### UI Component

**Role**: Web dashboard for GoClaw management

**Responsibilities**:
- Serve static web assets (HTML, CSS, JS)
- Provide interactive dashboard for managing providers, agents, resources
- WebSocket communication with server via `/ws` endpoint

**Ports**:
- `80` (HTTP) — Web UI

**Dependencies**:
- Server (via HTTP/WebSocket for API calls)
- No direct database access

**Resource Defaults**:
- Request: 50m CPU, 64Mi RAM
- Limit: 500m CPU, 128Mi RAM

**Probes**:
- Startup: `/health` on UI pod (nginx)
- Readiness: HTTP 200 on `/` or configured health path
- Liveness: HTTP 200 on `/` or configured health path

**Security**:
- UID 101 (nginx user)
- Read-only root filesystem: false (nginx needs /var/cache, /var/run writable)
- No privilege escalation
- All capabilities dropped

---

### PostgreSQL Component

**Role**: Persistent vector database with pgvector extension

**Responsibilities**:
- Store agent state, conversations, embeddings
- Support vector similarity search (pgvector)
- Provide ACID transactions for consistency

**Ports**:
- `5432` (TCP) — PostgreSQL protocol

**Dependencies**:
- None (self-contained)

**Resource Defaults**:
- Request: 100m CPU, 256Mi RAM
- Limit: 1000m CPU, 1Gi RAM

**Deployment Strategy**:
- `Recreate` (never RollingUpdate) to ensure single writer to PVC

**Storage**:
- `/var/lib/postgresql/data` — 20Gi PVC (ReadWriteOnce)
- Data persists across pod restarts

**Credentials**:
- User: `goclaw`
- Database: `goclaw`
- Password: Auto-generated (stored in Secret)

**Probes**:
- Startup: `pg_isready` every 3s, 10 failures (30s total)
- Readiness: `pg_isready` every 10s
- Liveness: `pg_isready` every 30s

**Security**:
- UID 999 (postgres system user)
- Read-only root filesystem
- No privilege escalation
- All capabilities dropped

---

### Chrome Sidecar Component

**Role**: Browser automation engine for web scraping and interaction

**Responsibilities**:
- Expose Chrome DevTools Protocol (CDP) on port 9222
- Execute browser automation commands (screenshot, navigate, interact)
- Support headless Chrome for performance

**Ports**:
- `9222` (WebSocket) — Chrome DevTools Protocol

**Dependencies**:
- None (self-contained)

**Resource Defaults**:
- Request: 100m CPU, 256Mi RAM
- Limit: 2000m CPU, 2Gi RAM

**Shared Memory**:
- `/dev/shm` — 2Gi emptyDir (required for Chrome multi-process)
- Default may be too small; increase if web automation fails

**Security**:
- UID 1000 (same as server pod, for compatibility)
- Read-only root filesystem
- No privilege escalation
- All capabilities dropped

---

### GCPlane Component

**Role**: GitOps control plane for declarative GoClaw resource management

**Responsibilities**:
- Watch manifest ConfigMap for changes
- Parse manifest YAML with variable substitution
- Continuously reconcile GoClaw resources (providers, agents, channels)
- Detect drift between desired (manifest) and actual (server) state
- Send notifications on drift via webhooks

**Ports**:
- `8480` (HTTP) — Readiness and metrics endpoints

**Dependencies**:
- GoClaw Server (HTTP API)
- Secret (GOCLAW_TOKEN)
- ConfigMap (manifest YAML)

**Resource Defaults**:
- Request: 50m CPU, 64Mi RAM
- Limit: 200m CPU, 128Mi RAM

**Configuration**:
- `RECONCILE_INTERVAL` — Frequency of manifest checks (default: 30s)
- `LOG_FORMAT` — "text" or "json" (default: json)
- `WEBHOOK_URL` — Optional drift notification endpoint
- `WEBHOOK_FORMAT` — slack, discord, teams, googlechat, or telegram

**Security**:
- UID 65534 (nobody user)
- Read-only root filesystem
- No privilege escalation
- All capabilities dropped

---

## Networking

### Service Discovery

**In-Cluster DNS** (Kubernetes CoreDNS):
```
Server:      goclaw.default.svc.cluster.local:18790
UI:          goclaw-ui.default.svc.cluster.local:80
Database:    goclaw-db.default.svc.cluster.local:5432
GCPlane:     goclaw-gcplane.default.svc.cluster.local:8480
```

### Ingress Routing

**Path-Based Routing** (when ingress.enabled: true):
```
Ingress Host: api.example.com

GET /                  → Service: goclaw-ui:80        (UI dashboard)
GET /v1/*              → Service: goclaw:18790         (Server REST API)
GET /ws                → Service: goclaw:18790         (Server WebSocket)
GET /health            → Service: goclaw:18790         (Server health)
```

### Network Policies (Optional)

If `networkPolicy.enabled: true`, restrict traffic:
```
Ingress:
- Allow external → goclaw:18790 (server)
- Allow external → goclaw-ui:80 (UI)
Egress:
- Allow goclaw pod → goclaw-db:5432 (database)
- Allow goclaw pod → external endpoints (LLM providers)
```

---

## Storage Architecture

### Persistent Volumes

| PVC | Size | Mode | Node Affinity | Use Case |
|-----|------|------|---------------|----------|
| **data** | 5Gi | RWO | None | GoClaw config, state, serialized data |
| **workspace** | 10Gi | RWO | None | Agent-generated artifacts, embeddings, cache |
| **db** | 20Gi | RWO | None | PostgreSQL data files, WAL logs, indexes |

**Access Mode**:
- **RWO** (ReadWriteOnce): Single pod mount only
  - Restricts to single-replica server (or requires node affinity)
  - For multi-replica, use external DB or RWX storage (NFS, shared filesystem)

**Storage Classes**:
- User-specified via `persistence.*.storageClass` (empty = cluster default)
- Common: EBS (AWS), AzureDisk (Azure), local (single-node), NFS (multi-node)

### Data Persistence Across Restarts

```
Pod Restart (health probe failure):
1. Pod terminated; PVC unmounted
2. New pod scheduled
3. PVC re-mounted to new pod
4. Data intact; no loss

Deployment Rolling Update:
1. Old pod: grace period (default 30s) for graceful shutdown
2. New pod: starts and mounts PVC
3. New pod initializes from persisted data
4. Old pod terminated after new pod ready

PVC Deletion Safeguards:
- Don't use 'helm uninstall' without backup
- PVCs are not auto-deleted (retain policy)
- Secrets protected with helm.sh/resource-policy: keep
```

---

## Security Model

### Pod Security Context

**Server Pod**:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

**Volumes**:
- `/tmp`, `/dev/shm` — writable (emptyDir)
- All others — read-only mounts from ConfigMap/Secret/PVC

### Secret Management

**Auto-Generated Secrets**:
```
config.existingSecret == "" (empty)
→ Create Secret with:
   - GOCLAW_GATEWAY_TOKEN: randAlphaNum(32)
   - GOCLAW_ENCRYPTION_KEY: randAlphaNum(32)
   - POSTGRES_PASSWORD: randAlphaNum(32)
```

**Existing Secret Mode**:
```
config.existingSecret == "my-secret"
→ Reference external secret (user-managed)
→ User responsible for rotation, backup, access control
```

**Secret Protection**:
- `helm.sh/resource-policy: keep` — prevents accidental deletion during `helm uninstall`
- Secrets not logged in pod events or Kubernetes audit logs (handled by kubelet)
- Access restricted to pod via RBAC (ServiceAccount bound to namespace)

### Image Pull Secrets

For private registries:
```yaml
global.imagePullSecrets:
  - name: ghcr-credentials
  - name: docker-hub-credentials
```

Secrets must exist in namespace before Helm install.

### Network Policies (Optional)

```yaml
networkPolicy.enabled: true
→ Allow ingress from Ingress Controller only
→ Allow egress to DNS, PostgreSQL, external services
→ Deny pod-to-pod communication by default
```

---

## Deployment Strategies

### Development/Single-Node

```yaml
server.replicas: 1
server.strategy.type: RollingUpdate
db.enabled: true
persistence.data.size: 5Gi
persistence.workspace.size: 10Gi
```

**Characteristics**:
- Single server pod on one node
- Sidecar PostgreSQL simplifies setup (no external DB required)
- RollingUpdate safe (RWO PVC stays on same node)
- All data in-cluster (no external dependencies)

### Staging/Multi-Replica (External DB)

```yaml
server.replicas: 3
server.strategy.type: RollingUpdate
db.enabled: false
externalDatabase.host: postgres.example.com
externalDatabase.port: 5432
```

**Characteristics**:
- 3 server pods distributed across nodes
- External PostgreSQL (must exist)
- RollingUpdate ensures zero downtime
- Load-balanced via Service

### Production (HA + Monitoring)

```yaml
server.replicas: 3
server.strategy.type: RollingUpdate
server.podDisruptionBudget.enabled: true
server.podDisruptionBudget.minAvailable: 1
server.resources.limits.memory: 2Gi
db.enabled: false
externalDatabase.host: cloudsql.example.com
gcplane.enabled: true
gcplane.webhook.url: https://hooks.slack.com/...
ingress.enabled: true
ingress.host: api.example.com
ingress.tls: true
```

**Characteristics**:
- HA with 3 replicas and PDB
- External managed PostgreSQL (Cloud SQL, RDS)
- GitOps automation via GCPlane
- TLS termination via Ingress
- Drift notifications to Slack

---

## Performance Considerations

### Bottlenecks

| Component | Bottleneck | Mitigation |
|-----------|-----------|-----------|
| **Server CPU** | AI processing, request handling | Increase `server.resources.limits.cpu` |
| **Server Memory** | Model inference, cache | Increase `server.resources.limits.memory` |
| **PostgreSQL CPU** | Query execution, indexing | Increase `db.resources.limits.cpu` |
| **PostgreSQL Storage** | Large embeddings, long conversation history | Increase PVC size, use external managed DB |
| **Network I/O** | External API calls, large transfers | Use private cluster egress, optimize payload size |
| **Chrome Sidecar** | Concurrent web automation | Increase `browser.resources.limits.cpu/memory` |

### Scaling Considerations

**Horizontal Scaling**:
- Increase `server.replicas` for load distribution
- Must use external PostgreSQL + RWX storage (NFS) for multi-node
- Load-balanced via Service; no sticky sessions required

**Vertical Scaling**:
- Increase CPU/memory limits in `resources.limits`
- May trigger pod eviction if node has insufficient resources
- Monitor cluster resource utilization

---

## Monitoring & Observability

### Health Checks

```bash
# Server readiness
kubectl get pod <pod-name> -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# Database connectivity
kubectl exec <server-pod> -- wget -q -O- http://localhost:18790/health

# GCPlane reconciliation status
kubectl exec <gcplane-pod> -- curl http://localhost:8480/readyz
```

### Logs

```bash
# Server logs
kubectl logs <server-pod> -f

# Database logs
kubectl logs <db-pod> -f

# GCPlane logs (JSON format)
kubectl logs <gcplane-pod> -f | jq
```

### Prometheus Metrics

GCPlane service exposes metrics on port 8480:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8480"
```

---

## High Availability (HA) Setup

**Requirements**:
1. External PostgreSQL (managed service recommended)
2. Multi-replica server deployment (≥ 3)
3. RollingUpdate strategy with maxUnavailable: 0
4. Pod Disruption Budget with minAvailable: ≥ 1
5. Load balancer or Ingress controller
6. Distributed node topology (anti-affinity optional but recommended)

**Failover Behavior**:
```
Server pod crash:
1. kubelet detects unhealthy pod (liveness probe timeout)
2. Pod terminated; traffic routed to remaining pods
3. ReplicaSet creates replacement pod
4. New pod mounts shared PVCs (if using RWX)

Node failure:
1. Kubernetes detects node unavailable (heartbeat timeout)
2. Pods evicted after grace period (5 min default)
3. ReplicaSet reschedules pods on healthy nodes
4. PVCs remounted on new nodes (if supported by storage class)
```

---

## Disaster Recovery (DR)

### Data Backup

**Critical Data**:
- PostgreSQL database (in DB PVC or external managed DB)
- Configuration and state in data PVC
- Secrets and encryption keys in Kubernetes Secret

**Backup Strategy**:
```
1. External managed PostgreSQL with automated backups (RDS, Cloud SQL, etc.)
   - Point-in-time recovery (PITR) enabled
   - Automated daily snapshots
   - Cross-region replication

2. PVC snapshots (via storage provider)
   - EBS snapshots, AzureDisk snapshots, GCS persistent disks
   - Automated daily snapshots with retention policy

3. Secret backups
   - Export Kubernetes Secrets to secure vault
   - Rotate credentials regularly
```

**Recovery Procedure**:
```
1. Restore PostgreSQL from backup (or from managed DB)
2. Restore PVCs from snapshots or re-provision empty volumes
3. Restore Secrets from vault
4. Re-deploy chart with same release name
5. Verify data integrity and health probes
```

### Restore from Backup

```bash
# Point-in-time recovery (PostgreSQL)
gsutil -m cp gs://backup-bucket/db-backup-2025-04-02.sql.gz ./
gunzip db-backup-2025-04-02.sql.gz
psql goclaw < db-backup-2025-04-02.sql

# Helm restore
helm install goclaw ./charts/goclaw \
  -f values-production.yaml \
  --set externalDatabase.host=restored-postgres.example.com
```



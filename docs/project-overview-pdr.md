# GoClaw Charts - Project Overview & PDR

## Project Overview

**GoClaw Charts** is a production-ready Helm chart for deploying **GoClaw**, a multi-agent AI gateway that orchestrates interactions between AI agents, language models, and external services. The chart provides a complete, containerized deployment solution with optional PostgreSQL database, web UI, browser automation sidecar, and GitOps control plane integration.

### Purpose
Enable organizations to deploy GoClaw on Kubernetes with minimal configuration while maintaining flexibility for development, staging, and production environments.

### Target Users
- DevOps engineers managing Kubernetes infrastructure
- Platform teams deploying multi-agent AI systems
- Organizations requiring enterprise-grade AI gateway deployments with GitOps automation

### Key Features
- **Modular Component Architecture**: Toggle server, UI, database, browser sidecar, and GitOps plane independently
- **Multi-Registry Support**: Deploy from GHCR, Docker Hub, or private registries
- **Image Variants**: Standard, API-only, full-runtime, or OpenTelemetry-instrumented server images
- **Secure Defaults**: Non-root containers, read-only filesystems, auto-generated secrets with protection policies
- **Production-Ready Database**: Sidecar PostgreSQL for dev/testing or external database for production
- **Web Automation**: Integrated Chrome sidecar for web scraping and automation via Chrome DevTools Protocol
- **GitOps Integration**: Optional GCPlane declarative control plane for continuous reconciliation of GoClaw resources
- **Zero-Downtime Upgrades**: Configurable deployment strategy with rolling updates
- **Health Probes**: Startup, readiness, and liveness probes for robust Kubernetes orchestration
- **Persistent Storage**: Configurable PVCs for data, workspace, and database with storage class support
- **Ingress Ready**: Path-based routing for server APIs, UI, and health endpoints

---

## Product Development Requirements (PDR)

### Functional Requirements

#### F1: Multi-Component Deployment
- **Requirement**: Chart must deploy up to 5 independent components via single Helm release
- **Components**: Server (required), UI (optional), PostgreSQL (optional), Chrome sidecar (optional), GCPlane (optional)
- **Toggle Mechanism**: Each component controllable via `values.yaml` boolean flags
- **Acceptance**: All combinations of enabled/disabled components must deploy without errors

#### F2: Database Flexibility
- **Requirement**: Support both sidecar and external database configurations
- **Sidecar Mode**: Deploy PostgreSQL with pgvector extension for vector embeddings
- **External Mode**: Connect to pre-existing PostgreSQL via DSN or connection parameters
- **Automatic DSN Construction**: Chart must build correct DSN from connection parameters when external URL not provided
- **Password Management**: Support both auto-generated and existing secrets
- **Acceptance**: DSN construction produces valid PostgreSQL connection strings; db connectivity verified via init container

#### F3: Image Registry & Variant Support
- **Requirement**: Deploy server image from multiple registries and variants
- **Registries**: GHCR (primary), Docker Hub, private registries via imagePullSecrets
- **Variants**: Standard, `-base` (API-only), `-full` (all runtimes), `-otel` (OpenTelemetry)
- **Tag Override**: Chart defaults to `v2.65.0-full`; support custom versions via values
- **Acceptance**: All registry/variant combinations pull and start correctly

#### F4: Security by Default
- **Requirement**: All containers run with non-root UID and restrictive capabilities
- **UIDs**: Server/Chrome uid 1000, PostgreSQL uid 999, GCPlane uid 65534
- **Filesystems**: Read-only root except UI; writable tmpfs for temp files
- **Capabilities**: All dropped; no privilege escalation
- **Secrets**: Auto-generate gateway token, encryption key, DB password; protect with helm.sh/resource-policy: keep
- **Acceptance**: `kubectl auth can-i` checks verify no privilege escalation; secrets readable only by chart namespace

#### F5: Ingress & Service Networking
- **Requirement**: Expose server, UI, and GCPlane via Services and optional Ingress
- **Services**: ClusterIP for server (18790), UI (80), database (5432 internal-only), GCPlane (8480)
- **Ingress**: Path-based routing: `/ws`, `/v1`, `/health` → server; `/` → UI
- **DNS**: Support custom hostname with TLS termination
- **Acceptance**: All services resolvable in-cluster; ingress routes traffic correctly to backend pods

#### F6: Persistent Storage Management
- **Requirement**: Allocate and manage PVCs for data, workspace, and database
- **Data PVC**: 5Gi default, stores GoClaw configuration and state
- **Workspace PVC**: 10Gi default, stores agent-generated artifacts
- **Database PVC**: 20Gi default, stores PostgreSQL data
- **Storage Classes**: Support custom storage class per PVC; use cluster default if unspecified
- **Acceptance**: PVCs provisioned; data persists across pod restarts

#### F7: GCPlane GitOps Integration
- **Requirement**: Optional GCPlane deployment for declarative GoClaw resource management
- **Manifest Support**: User-provided or auto-generated manifest defining providers, agents, channels
- **Token Injection**: Inject GOCLAW_TOKEN from chart secret into GCPlane environment
- **Reconciliation**: Configurable interval (default 30s) for manifest reconciliation
- **Webhooks**: Optional drift notification webhooks (Slack, Discord, Teams, Google Chat, Telegram)
- **Acceptance**: GCPlane reconciles manifest; tokens injected correctly; webhooks fire on drift

#### F8: Health Probes & Readiness
- **Requirement**: Configure startup, readiness, and liveness probes for server
- **Startup Probe**: Allows 30s grace period (10 failures × 3s period) before marking unhealthy
- **Readiness Probe**: Checks `/health` every 10s; 3s timeout
- **Liveness Probe**: Checks `/health` every 30s; 3s timeout
- **Database Wait**: Init container blocks server start until PostgreSQL ready (TCP port 5432)
- **Acceptance**: Unhealthy pods are not routed traffic; killed containers restarted correctly

#### F9: Resource Management
- **Requirement**: Define CPU/memory requests and limits for each component
- **Server**: 100m/256Mi request, 2000m/1Gi limit
- **UI**: 50m/64Mi request, 500m/128Mi limit
- **Database**: 100m/256Mi request, 1000m/1Gi limit
- **Browser**: 100m/256Mi request, 2000m/2Gi limit
- **GCPlane**: 50m/64Mi request, 200m/128Mi limit
- **Acceptance**: All pods scheduled; memory and CPU limits enforced; OOMKilled pods logged

#### F10: Pod Disruption Budgets
- **Requirement**: Optional PDB for server and UI to prevent cluster disruptions
- **Default**: Disabled; can enable with `minAvailable: 1` or `maxUnavailable: 1`
- **Acceptance**: When enabled, PDB blocks eviction of required replicas

### Non-Functional Requirements

#### NF1: Helm Chart Standards
- **Chart Version**: v0.2.0; App Version v2.65.0
- **API Version**: v2 (Helm 3+)
- **Template Language**: Go templating with Helm functions
- **Helper Functions**: 9 reusable functions in `_helpers.tpl` for name, labels, database config
- **Acceptance**: `helm lint` passes; `helm template` produces valid YAML; `helm install/upgrade` succeeds

#### NF2: Documentation Quality
- **README**: Comprehensive guide with prerequisites, quickstart, advanced configuration, troubleshooting
- **Values Defaults**: All chart values documented inline with YAML comments
- **Examples**: Real-world deployment examples (dev, staging, production, GCPlane setup)
- **Acceptance**: First-time users can deploy without external documentation

#### NF3: Configuration Flexibility
- **Requirement**: Support custom resource requests, replicas, storage, environment variables, labels, annotations
- **Override Mechanism**: Standard Helm `-f values.yaml` and `--set` key=value patterns
- **Defaults**: Sensible defaults for dev/testing; documented production recommendations
- **Acceptance**: All configuration options in values.yaml can be overridden; changes reflected in deployed manifests

#### NF4: Version Compatibility
- **Kubernetes**: 1.24+ (required for security policies, service discovery)
- **Helm**: 3.0+ (API v2 only)
- **PostgreSQL**: 14+ with pgvector extension (for sidecar mode)
- **Acceptance**: Chart installs successfully on specified Kubernetes/Helm versions

#### NF5: Backward Compatibility
- **Requirement**: Maintain upgrade path for existing deployments
- **Breaking Changes**: Documented with migration steps in changelog
- **Resource Names**: Stable; no breaking renames that orphan existing PVCs/secrets
- **Acceptance**: `helm upgrade` succeeds without data loss for v0.1.x → v0.2.0

#### NF6: Observability & Troubleshooting
- **Requirement**: Support debugging and monitoring of chart deployments
- **Logs**: Container logs accessible via `kubectl logs`
- **Events**: Pod and resource events visible in `kubectl describe`
- **Health**: Probes and init containers indicate startup/readiness status
- **Post-Install Notes**: NOTES.txt provides access instructions and GCPlane status check commands
- **Acceptance**: Issues diagnosable from standard `kubectl` commands

### Success Metrics

| Metric | Target | Validation |
|--------|--------|-----------|
| **Installation Time** | < 2 minutes | Measure helm install duration |
| **Helm Lint Passes** | 100% | Zero warnings/errors on `helm lint` |
| **Template Validation** | 100% | `helm template` produces valid YAML |
| **Security Compliance** | 100% | All containers non-root; read-only filesystems except UI |
| **Database Connectivity** | 100% | Init container successfully waits for DB; app connects without errors |
| **Component Toggle** | All combinations | Enable/disable any component; deployment stable |
| **Documentation Coverage** | 100% | All values documented; examples cover dev/staging/prod/gcplane |
| **Upgrade Success Rate** | 100% | v0.1.x → v0.2.0 with zero data loss |

---

## Technical Constraints

1. **Helm 3.0+**: No Helm 2 support; v2 API only
2. **Kubernetes 1.24+**: Pod Security Policy replaced by Pod Security Standards; RBAC required
3. **Single Namespace**: Chart deploys all resources to a single namespace per release
4. **RWO PVC Limitation**: Data and workspace PVCs use ReadWriteOnce; multi-replica server requires RWX storage class
5. **External Database Required for HA**: Sidecar PostgreSQL (Recreate strategy) incompatible with multi-replica deployments; use external DB + RollingUpdate strategy for HA
6. **Chrome Sidecar Resource Hungry**: Default 2Gi memory limit; may need increase for high-concurrency web automation
7. **GCPlane Token Injection**: GOCLAW_TOKEN secret must exist; GCPlane init container fails if secret missing
8. **Ingress Annotation Support**: Some ingress controllers require specific annotations (e.g., cert-manager for TLS)

---

## Dependencies & External Services

- **Kubernetes 1.24+**: Container orchestration platform
- **PostgreSQL 14+** (sidecar or external): Vector database for embeddings
- **Image Registries**: GHCR, Docker Hub (network access required)
- **Chrome Binary** (in browser sidecar): Alpine Linux base required
- **Ingress Controller**: Optional (nginx, Traefik, etc.)
- **Storage Provisioner**: Required for dynamic PVC provisioning (EBS, AzureDisk, local storage)
- **External GoClaw Services**: (optional) External API endpoints, LLM providers, data sources

---

## Out of Scope

- **Namespace Management**: Chart does not create namespaces; user responsibility
- **RBAC Configuration**: Chart does not create ServiceAccounts or ClusterRoles; assumes user has full namespace permissions
- **TLS Certificate Management**: Ingress TLS certificates managed externally (e.g., cert-manager)
- **Backup & Disaster Recovery**: PVC backup strategies left to user/storage administrator
- **Multi-Cluster Deployment**: Single-cluster scope; multi-cluster setups handled by user tooling
- **GitOps Workflow Automation**: GCPlane manifest authoring and GitOps CI/CD pipelines outside chart scope

---

## Acceptance Criteria

The chart is production-ready when:

1. [ ] All helm lint checks pass without warnings
2. [ ] Template rendering produces valid YAML for all component combinations
3. [ ] `helm install` succeeds on Kubernetes 1.24+ with default values
4. [ ] All containers run as non-root with read-only root filesystem (except UI)
5. [ ] Database init container waits for PostgreSQL connectivity before server starts
6. [ ] PVCs are provisioned and data persists across pod restarts
7. [ ] Server, UI, and GCPlane endpoints are accessible via Services/Ingress
8. [ ] Secrets are auto-generated with protection policy; sensitive data not logged
9. [ ] Health probes function correctly; unhealthy pods removed from service
10. [ ] Upgrade from v0.1.x to v0.2.0 succeeds without data loss
11. [ ] Documentation is complete and examples are tested
12. [ ] No critical security issues found in dependency scanning

---

## Version History

| Version | Release Date | Status |
|---------|-------------|--------|
| v0.2.0 | 2025-04-03 | Current |
| v0.1.0 | 2025-03-15 | Released |


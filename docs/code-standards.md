# GoClaw Charts - Code Standards

## Overview

This document defines coding standards, patterns, and best practices for the GoClaw Helm chart repository. These standards ensure consistency, maintainability, and security across all templates and configurations.

---

## Chart Organization

### File Naming Conventions

**Helm Templates**:
- Use lowercase with hyphens: `deployment.yaml`, `service.yaml`, `configmap-gcplane.yaml`
- Descriptive names: `deployment-ui.yaml` (not `deploy-ui.yaml`)
- Component-specific: `service-db.yaml` (database service, not generic `service2.yaml`)
- Helper templates: `_helpers.tpl` (leading underscore by convention)
- Post-install notes: `NOTES.txt` (uppercase by convention)

**Values & Config**:
- `values.yaml` — default configuration
- `values-dev.yaml`, `values-prod.yaml` — environment-specific overrides

**Documentation**:
- `README.md` — chart overview and quickstart
- `Chart.yaml` — metadata (no template logic here)

---

## YAML Structure & Formatting

### Indentation

- **2 spaces** per indentation level (Helm convention)
- Never use tabs
- Consistent indentation within sections

**Good**:
```yaml
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: goclaw
```

**Bad**:
```yaml
spec:
    replicas: 3  # 4 spaces (wrong)
  template:
    metadata:
      labels:
        app: goclaw
```

### Line Length

- Keep lines under **100 characters** when practical
- Break long strings with pipes (`|`) for readability
- Use Helm functions to reduce verbosity

**Good**:
```yaml
env:
  - name: GOCLAW_POSTGRES_DSN
    value: |
      postgres://{{ .Values.db.user }}:{{ .Values.db.password }}@{{ .Values.db.host }}:{{ .Values.db.port }}/{{ .Values.db.name }}?sslmode=disable
```

**Less Ideal**:
```yaml
env:
  - name: GOCLAW_POSTGRES_DSN
    value: postgres://{{ .Values.db.user }}:{{ .Values.db.password }}@{{ .Values.db.host }}:{{ .Values.db.port }}/{{ .Values.db.name }}?sslmode=disable
```

---

## Helm Template Conventions

### Template Syntax

**Spacing Around Brackets**:
```yaml
{{ include "goclaw.fullname" . }}      # Correct
{{include "goclaw.fullname" .}}        # Avoid (no spacing)
{{ include "goclaw.fullname" .}}       # Avoid (inconsistent)
```

**Conditional Blocks**:
```yaml
{{ if .Values.db.enabled }}
# content here
{{ end }}

{{ if .Values.db.enabled }}
# content
{{ else }}
# alternative
{{ end }}

{{ if and .Values.server.enabled .Values.ui.enabled }}
# both enabled
{{ end }}
```

**Whitespace Control** (when needed):
```yaml
{{- if .Values.db.enabled }}  # Remove leading whitespace
# content
{{- end }}                     # Remove trailing whitespace
```

### Comments

**Inline Comments** (sparingly):
```yaml
replicas: {{ .Values.server.replicas }}  # Number of server pods
```

**Section Headers**:
```yaml
# --- Server Configuration ---
# Defines API server deployment settings
```

**Explaining Complex Logic**:
```yaml
{{ if .Values.db.enabled }}
# Sidecar database: use fixed credentials
- name: POSTGRES_HOST
  value: {{ include "goclaw.dbHost" . }}
{{ else if .Values.externalDatabase.url }}
# External database: use full DSN
- name: GOCLAW_POSTGRES_DSN
  value: {{ .Values.externalDatabase.url }}
{{ else }}
# External database: construct DSN from parameters
- name: GOCLAW_POSTGRES_DSN
  value: "postgres://{{ .Values.externalDatabase.user }}:{{ .POSTGRES_PASSWORD }}@..."
{{ end }}
```

---

## Helper Function Standards

### Naming Convention

**Pattern**: `{{ define "goclaw.<function-name>" -}}`
- Lowercase with dots as separators
- Scoped to chart name: `goclaw.name`, `goclaw.fullname`, `goclaw.labels`
- No hyphens in function names (Helm convention)

**Existing Helpers**:
```
goclaw.name               # Chart name (default .Chart.Name)
goclaw.fullname          # Full release name
goclaw.labels            # Standard Kubernetes labels
goclaw.selectorLabels    # Pod selector labels
goclaw.dbHost            # Database hostname (sidecar or external)
goclaw.dbPort            # Database port
goclaw.dbName            # Database name
goclaw.dbUser            # Database user
goclaw.secretName        # Secret reference (existing or auto-generated)
```

### Function Documentation

**Header Comments**:
```yaml
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "goclaw.fullname" -}}
...
{{- end }}
```

### Function Implementation Rules

1. **Idempotence**: Same inputs → Same output (no side effects)
2. **Determinism**: No random values in helpers
3. **Error Handling**: Fail fast if inputs invalid
4. **Reusability**: Avoid dependencies on other helpers if possible
5. **Output Format**: Quoted if output is YAML key/value, unquoted if for shell/env

**Example: Good Helper**:
```yaml
{{- define "goclaw.dbHost" -}}
{{- if .Values.db.enabled }}
{{- printf "%s-db" (include "goclaw.fullname" .) }}
{{- else }}
{{- .Values.externalDatabase.host }}
{{- end }}
{{- end }}
```

---

## Values.yaml Standards

### Organization Structure

```yaml
# Global settings shared across all components
global:
  imagePullSecrets: []

# Server image configuration
image:
  repository: ghcr.io/nextlevelbuilder/goclaw
  tag: "v2.65.0-full"
  pullPolicy: IfNotPresent

# Server deployment
server:
  replicas: 1
  port: 18790
  # ... nested options

# UI deployment (optional)
ui:
  enabled: true
  # ... nested options

# Database (optional)
db:
  enabled: true
  # ... nested options

# External database (when db.enabled=false)
externalDatabase:
  url: ""
  host: ""
  # ... nested options

# Persistent storage
persistence:
  data:
    enabled: true
    size: 5Gi
  workspace:
    enabled: true
    size: 10Gi

# Ingress (optional)
ingress:
  enabled: false
  # ... nested options

# Health probes
probes:
  startup: {}
  readiness: {}
  liveness: {}

# Browser sidecar (optional)
browser:
  enabled: true
  # ... nested options

# Secrets
config:
  gatewayToken: ""
  encryptionKey: ""
  existingSecret: ""

# GCPlane GitOps control plane (optional)
gcplane:
  enabled: false
  # ... nested options
```

### Documentation Comments

**Format**: YAML comment above the key (or inline for single values)

```yaml
# -- Brief description of what this setting does
# Longer explanation if needed (multi-line)
# Example: somePath.in.values: value
key: defaultValue
```

**Example**:
```yaml
# -- Number of replicas for GoClaw server
# Increase for load distribution; requires external PostgreSQL for multi-replica
server:
  replicas: 1

# -- Server API port
# Maps to container port 18790
  port: 18790

# -- Deployment strategy: RollingUpdate or Recreate
# RollingUpdate: safe for RWO PVCs on single node
# Recreate: required for multi-node clusters with shared PVCs
  strategy:
    type: RollingUpdate

  # -- CPU and memory requests/limits
  resources:
    requests:
      # -- CPU request in millicores (100m = 0.1 CPU)
      cpu: 100m
      # -- Memory request in Mi (256Mi = 256 megabytes)
      memory: 256Mi
    limits:
      cpu: "2"
      memory: 1Gi
```

### Conditional Values

**Pattern**: Use boolean flags to enable/disable features
```yaml
# Feature disabled by default; user must explicitly enable
ui:
  enabled: false     # Explicitly set by values override
  image: ...
  # ... other UI config only loaded if enabled is true
```

### Sensitive Values

**Pattern**: Empty defaults; user provides values
```yaml
config:
  # -- API gateway token (auto-generated if empty)
  gatewayToken: ""
  
  # -- Encryption key for sensitive data (auto-generated if empty)
  encryptionKey: ""
  
  # -- Reference to existing secret (bypasses auto-generation)
  existingSecret: ""
```

---

## Template Patterns

### Conditional Rendering (Component Enable/Disable)

**Pattern**:
```yaml
{{ if .Values.ui.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "goclaw.fullname" . }}-ui
...
{{ end }}
```

**Usage**: Allows users to toggle components via `--set ui.enabled=false`

### Database Configuration

**Pattern**: Support both sidecar and external database
```yaml
{{ if .Values.db.enabled }}
# Sidecar database: use fixed values
- name: POSTGRES_HOST
  value: {{ include "goclaw.dbHost" . }}
- name: POSTGRES_PORT
  value: "5432"
{{ else if .Values.externalDatabase.url }}
# External database: user provides full DSN
- name: GOCLAW_POSTGRES_DSN
  value: {{ .Values.externalDatabase.url | quote }}
{{ else }}
# External database: construct DSN from parameters
- name: GOCLAW_POSTGRES_DSN
  value: "postgres://{{ .Values.externalDatabase.user }}:{{ $password }}@{{ .Values.externalDatabase.host }}:{{ .Values.externalDatabase.port }}/{{ .Values.externalDatabase.name }}?sslmode=disable"
{{ end }}
```

### Secret Reference

**Pattern**: Use existing secret or auto-generated
```yaml
secretKeyRef:
  name: {{ include "goclaw.secretName" . }}
  key: POSTGRES_PASSWORD
```

This helper function checks `config.existingSecret` and returns appropriate secret name.

### Init Containers

**Pattern**: Wait for dependencies before main container
```yaml
{{ if .Values.db.enabled }}
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z {{ include "goclaw.fullname" . }}-db 5432; do echo waiting for db; sleep 2; done']
{{ end }}
```

### Linting with nindent

**Pattern**: Properly indent multi-line YAML blocks
```yaml
spec:
  template:
    metadata:
      labels:
        {{- include "goclaw.labels" . | nindent 8 }}
    spec:
      containers:
        - name: server
          resources:
            {{- toYaml .Values.server.resources | nindent 12 }}
```

**Why**: `nindent` ensures correct indentation in Go templates (8 spaces for labels, 12 for nested config).

---

## Security Standards

### Pod Security Context

**Principle**: Non-root, read-only filesystem, no privilege escalation

**Pattern**:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000           # Application UID
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:           # Optional
    type: RuntimeDefault
```

### Container Security Context

**Pattern**: Inherit from pod; override only if necessary
```yaml
containers:
  - name: server
    securityContext:
      runAsUser: 1000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

### Secret Handling

**Pattern 1 — Auto-Generate**:
```yaml
data:
  GOCLAW_GATEWAY_TOKEN: {{ randAlphaNum 32 | b64enc | quote }}
  POSTGRES_PASSWORD: {{ randAlphaNum 32 | b64enc | quote }}
```

**Pattern 2 — Reference Existing**:
```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ include "goclaw.secretName" . }}
        key: POSTGRES_PASSWORD
```

**Pattern 3 — Protect from Deletion**:
```yaml
metadata:
  annotations:
    helm.sh/resource-policy: keep
```

### Image Pull Secrets

**Pattern**:
```yaml
{{ with .Values.global.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 2 }}
{{ end }}
```

**Usage**:
```bash
helm install goclaw ./charts/goclaw \
  --set global.imagePullSecrets[0].name=ghcr-secret
```

---

## Labels & Selectors

### Standard Labels

**Pattern** (via helper):
```yaml
{{- include "goclaw.labels" . | nindent 4 }}
```

**Output**:
```yaml
helm.sh/chart: goclaw-0.2.0
app.kubernetes.io/name: goclaw
app.kubernetes.io/instance: my-release
app.kubernetes.io/version: "v2.65.0"
app.kubernetes.io/managed-by: Helm
```

### Selector Labels

**Pattern** (via helper):
```yaml
selector:
  matchLabels:
    {{- include "goclaw.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: server
```

**Output**:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: goclaw
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/component: server
```

### Component-Specific Labels

**Pattern**:
```yaml
labels:
  {{- include "goclaw.labels" . | nindent 4 }}
  app.kubernetes.io/component: server    # Identifies which component
```

---

## Resource Definitions

### Resource Requests & Limits

**Pattern**:
```yaml
resources:
  requests:
    cpu: 100m         # Minimum guaranteed
    memory: 256Mi
  limits:
    cpu: "2"          # Hard ceiling
    memory: 1Gi       # Hard ceiling (OOMKill if exceeded)
```

**Defaults** (production-ready):
- Server: 100m/256Mi request, 2000m/1Gi limit
- UI: 50m/64Mi request, 500m/128Mi limit
- Database: 100m/256Mi request, 1000m/1Gi limit
- Browser: 100m/256Mi request, 2000m/2Gi limit
- GCPlane: 50m/64Mi request, 200m/128Mi limit

---

## Health Probes

### Startup Probe

**Pattern**:
```yaml
startupProbe:
  httpGet:
    path: {{ .Values.probes.startup.path }}
    port: http
  initialDelaySeconds: {{ .Values.probes.startup.initialDelay }}
  periodSeconds: {{ .Values.probes.startup.period }}
  failureThreshold: {{ .Values.probes.startup.failureThreshold }}
```

**Purpose**: Allow container extra time to initialize without being marked unhealthy

### Readiness Probe

**Pattern**:
```yaml
readinessProbe:
  httpGet:
    path: {{ .Values.probes.readiness.path }}
    port: http
  periodSeconds: {{ .Values.probes.readiness.period }}
  timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
```

**Purpose**: Prevent traffic routing to pods not ready to serve requests

### Liveness Probe

**Pattern**:
```yaml
livenessProbe:
  httpGet:
    path: {{ .Values.probes.liveness.path }}
    port: http
  periodSeconds: {{ .Values.probes.liveness.period }}
  timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
```

**Purpose**: Restart pods that are stuck or deadlocked

---

## Testing & Validation

### Helm Linting

```bash
helm lint ./charts/goclaw
```

**Checks**:
- YAML syntax
- Chart structure
- Required fields (Chart.yaml, appVersion)
- Template validity

### Template Rendering

```bash
helm template my-release ./charts/goclaw -f values-prod.yaml
```

**Validation**:
- Verify all conditional logic renders correctly
- Check variable substitution
- Ensure no template syntax errors

### Dry-Run Install

```bash
helm install my-release ./charts/goclaw --dry-run --debug
```

**Validation**:
- Check resource creation logic
- Verify dependencies resolved
- Validate naming conventions

---

## Code Review Checklist

Before merging chart changes:

- [ ] Helm lint passes
- [ ] Template rendering valid for all component combinations
- [ ] Security context applied (non-root, read-only, no escalation)
- [ ] Health probes defined for stateful components
- [ ] Resource requests/limits reasonable for component
- [ ] All values documented with inline comments
- [ ] Conditional logic uses correct helpers
- [ ] Labels follow Kubernetes conventions
- [ ] Secret handling doesn't log sensitive data
- [ ] Breaking changes documented in CHANGELOG
- [ ] Examples updated if values schema changed

---

## Common Pitfalls

### 1. Inconsistent Indentation

**Problem**: Mixed spaces/tabs breaks YAML parsing
**Solution**: Use 2-space indentation consistently; enable editor settings

### 2. Missing `toYaml` with `nindent`

**Problem**: Multi-line values misaligned
```yaml
# Wrong
resources: {{ toYaml .Values.server.resources }}

# Correct
resources:
  {{- toYaml .Values.server.resources | nindent 2 }}
```

### 3. Unquoted Numeric Values

**Problem**: `port: 18790` (number) vs `port: "18790"` (string) type mismatch
**Solution**: Quote all template outputs: `port: {{ .Values.server.port | quote }}`

### 4. Hardcoded Values in Templates

**Problem**: Values not in values.yaml can't be overridden
**Solution**: Always move configuration to values.yaml with defaults

### 5. No Linting Before Commit

**Problem**: `helm install` fails due to template syntax errors
**Solution**: Run `helm lint` before committing; add to CI/CD

### 6. Forgetting Secret Protection

**Problem**: Secrets deleted during `helm uninstall`
**Solution**: Add `helm.sh/resource-policy: keep` to secrets

---

## Naming Conventions Summary

| Entity | Convention | Example |
|--------|-----------|---------|
| Chart | lowercase | `goclaw` |
| Template file | kebab-case | `deployment-ui.yaml` |
| Helper function | dot-separated, lowercase | `goclaw.fullname` |
| Kubernetes label | dots, slashes | `app.kubernetes.io/name` |
| Environment variable | UPPER_SNAKE_CASE | `GOCLAW_PORT` |
| ConfigMap key | UPPER_SNAKE_CASE | `GOCLAW_BROWSER_URL` |
| Secret key | UPPER_SNAKE_CASE | `POSTGRES_PASSWORD` |
| Release name | lowercase, alphanumeric | `my-goclaw-prod` |


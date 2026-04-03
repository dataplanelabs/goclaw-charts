# GoClaw Charts - Deployment Guide

## Prerequisites

### Kubernetes Cluster

- **Version**: 1.24 or later
- **Resources**: Minimum 2 CPU, 4Gi RAM (development); 8 CPU, 16Gi RAM (production)
- **Storage**: Dynamic volume provisioner configured (local-path, EBS, AzureDisk, etc.)
- **Networking**: Ingress controller installed (optional, for external access)

### Helm

- **Version**: 3.0 or later
- **Installation**: https://helm.sh/docs/intro/install/

Verify installation:
```bash
helm version
# Output: version.BuildInfo{Version:"v3.x.x", ...}
```

### Kubectl

- **Version**: 1.24 or later (matched to cluster version)

Verify cluster access:
```bash
kubectl cluster-info
kubectl get nodes
```

### Docker/Container Runtime

- **Docker**, **containerd**, or compatible runtime
- **Image Registry Access**:
  - Public: GHCR (`ghcr.io/nextlevelbuilder/`), Docker Hub (`digitop/`)
  - Private: Create image pull secrets if using private registries

---

## Quick Start (Development)

### 1. Add the Chart Repository (if using Helm repo)

Currently, this chart is not in a public Helm repository. Clone or download the chart locally:

```bash
git clone https://github.com/dataplanelabs/goclaw-charts.git
cd goclaw-charts
```

### 2. Deploy with Default Values

```bash
helm install my-goclaw ./charts/goclaw \
  --namespace goclaw \
  --create-namespace
```

**What This Does**:
- Creates namespace `goclaw`
- Deploys GoClaw server (1 replica)
- Deploys web UI (nginx)
- Deploys sidecar PostgreSQL with pgvector
- Deploys Chrome browser sidecar
- Auto-generates secrets (gateway token, encryption key, DB password)
- Creates PVCs for data (5Gi), workspace (10Gi), database (20Gi)

**Verify Installation**:
```bash
kubectl get all -n goclaw
kubectl get pvc -n goclaw
kubectl get secret -n goclaw
```

### 3. Access GoClaw

**Via Port-Forward** (development):
```bash
# API server
kubectl port-forward -n goclaw svc/my-goclaw 18790:18790

# Web UI
kubectl port-forward -n goclaw svc/my-goclaw-ui 8080:80

# In another terminal, access:
curl http://localhost:18790/health
open http://localhost:8080
```

**Via Service Name** (in-cluster):
```bash
kubectl run -it --rm --image=curlimages/curl --restart=Never -- \
  curl http://my-goclaw.goclaw.svc.cluster.local:18790/health
```

### 4. Retrieve Auto-Generated Secrets

```bash
# Gateway token
kubectl get secret -n goclaw my-goclaw -o jsonpath='{.data.GOCLAW_GATEWAY_TOKEN}' | base64 -d

# Database password
kubectl get secret -n goclaw my-goclaw -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d

# Encryption key
kubectl get secret -n goclaw my-goclaw -o jsonpath='{.data.GOCLAW_ENCRYPTION_KEY}' | base64 -d
```

### 5. Uninstall

```bash
helm uninstall my-goclaw -n goclaw
```

**Warning**: Persistent data (PVCs) and secrets are retained by default. To delete:
```bash
kubectl delete pvc --all -n goclaw
kubectl delete secret my-goclaw -n goclaw
kubectl delete namespace goclaw
```

---

## Staging Deployment (Multi-Replica, External DB)

### 1. Create Values Override File

Create `values-staging.yaml`:

```yaml
server:
  replicas: 3                    # Multiple replicas for load distribution
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # Zero downtime updates
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 1Gi

ui:
  enabled: true
  replicas: 2

db:
  enabled: false                 # Use external database

externalDatabase:
  host: postgres.staging.example.com
  port: 5432
  name: goclaw_staging
  user: goclaw
  existingSecret: db-credentials # Secret with POSTGRES_PASSWORD key

persistence:
  data:
    size: 20Gi
    storageClass: gp2            # AWS EBS gp2
  workspace:
    size: 50Gi
    storageClass: gp2

ingress:
  enabled: true
  host: api-staging.example.com
  className: nginx               # Specify your ingress controller
  tls: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"

browser:
  enabled: true
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
```

### 2. Create Database Credentials Secret

```bash
kubectl create secret generic db-credentials \
  -n goclaw \
  --from-literal=POSTGRES_PASSWORD='your-db-password'
```

### 3. Deploy to Staging

```bash
helm install goclaw-staging ./charts/goclaw \
  -f values-staging.yaml \
  -n goclaw-staging \
  --create-namespace
```

### 4. Verify Multi-Replica Deployment

```bash
kubectl get pods -n goclaw-staging
# Should show 3 server pods, 2 UI pods

kubectl get svc -n goclaw-staging
# Should show all services

kubectl describe ingress -n goclaw-staging
# Should show routing rules
```

### 5. Test Load Balancing

```bash
# Generate requests to multiple replicas
for i in {1..10}; do
  kubectl port-forward -n goclaw-staging svc/goclaw-staging $((18790 + i)):18790 &
done

# Make requests
curl http://localhost:18791/health
curl http://localhost:18792/health
curl http://localhost:18793/health
```

---

## Production Deployment (HA + GitOps)

### 1. Create Production Values

Create `values-production.yaml`:

```yaml
global:
  imagePullSecrets:
    - name: ghcr-credentials     # Private registry access

image:
  repository: ghcr.io/nextlevelbuilder/goclaw
  tag: "v2.65.0-full"            # Pinned version for reproducibility
  pullPolicy: IfNotPresent

server:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  podDisruptionBudget:
    enabled: true
    minAvailable: 1              # Keep at least 1 pod running
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 2Gi

ui:
  enabled: true
  replicas: 3
  podDisruptionBudget:
    enabled: true
    minAvailable: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: "1"
      memory: 256Mi

db:
  enabled: false                 # Use managed PostgreSQL

externalDatabase:
  host: cloudsql-instance.c.project.internal  # Cloud SQL private IP
  port: 5432
  name: goclaw_prod
  user: goclaw
  existingSecret: prod-db-secret

persistence:
  data:
    enabled: true
    size: 100Gi
    storageClass: fast-ssd        # High-performance storage
  workspace:
    enabled: true
    size: 200Gi
    storageClass: fast-ssd

ingress:
  enabled: true
  host: api.goclaw.example.com
  className: nginx
  tls: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

browser:
  enabled: true
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: "2"
      memory: 2Gi

config:
  existingSecret: goclaw-secrets  # User-managed secrets

gcplane:
  enabled: true                    # Enable GitOps automation
  replicas: 1
  interval: "30s"
  logFormat: "json"
  webhook:
    url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
    format: slack
  manifest: |
    apiVersion: gcplane/v1
    kind: Manifest
    metadata:
      name: goclaw-production
    spec:
      provider:
        url: http://goclaw-production.default.svc.cluster.local:18790
      resources:
        # Your GoClaw resources (providers, agents, channels) defined here
        # Example:
        - kind: Provider
          name: anthropic
          spec:
            type: anthropic
            apiKey: ${GOCLAW_TOKEN}
```

### 2. Create Production Secrets

```bash
# Database credentials
kubectl create secret generic prod-db-secret \
  -n goclaw-prod \
  --from-literal=POSTGRES_PASSWORD='$(openssl rand -base64 32)'

# GoClaw configuration (gateway token, encryption key)
kubectl create secret generic goclaw-secrets \
  -n goclaw-prod \
  --from-literal=GOCLAW_GATEWAY_TOKEN='$(openssl rand -base64 32)' \
  --from-literal=GOCLAW_ENCRYPTION_KEY='$(openssl rand -base64 32)'
```

### 3. Setup Image Pull Secret (GHCR)

```bash
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PAT \
  --docker-email=your@email.com \
  -n goclaw-prod
```

### 4. Deploy Production Release

```bash
helm install goclaw-prod ./charts/goclaw \
  -f values-production.yaml \
  -n goclaw-prod \
  --create-namespace \
  --wait \
  --timeout=10m
```

### 5. Verify Production HA Setup

```bash
# Check replicas
kubectl get deployments -n goclaw-prod
# Expected: 3 server replicas, 3 UI replicas, 1 GCPlane replica

# Check PDB
kubectl get pdb -n goclaw-prod
# Expected: 2 PDBs (server, UI)

# Check ingress
kubectl get ingress -n goclaw-prod
# Expected: Ingress with TLS and hostname

# Check GCPlane status
kubectl port-forward -n goclaw-prod svc/goclaw-prod-gcplane 8480:8480
curl http://localhost:8480/readyz
```

### 6. Monitor and Maintain

```bash
# Watch pod status
kubectl get pods -n goclaw-prod -w

# Check logs
kubectl logs -n goclaw-prod -l app.kubernetes.io/name=goclaw -f

# Monitor resource usage
kubectl top pods -n goclaw-prod
kubectl top nodes

# Check PVC usage
kubectl get pvc -n goclaw-prod

# Verify health probes
kubectl describe pod -n goclaw-prod <pod-name>
```

---

## GCPlane Setup (GitOps Automation)

### Enable GCPlane in Your Release

Update values to include:
```yaml
gcplane:
  enabled: true
  interval: "30s"           # Reconcile every 30 seconds
  logFormat: "json"         # JSON logs for structured logging
  webhook:
    url: https://hooks.slack.com/services/YOUR/WEBHOOK
    format: slack
  manifest: |
    apiVersion: gcplane/v1
    kind: Manifest
    metadata:
      name: my-goclaw
    spec:
      provider:
        url: http://goclaw.default.svc.cluster.local:18790
      resources:
        - kind: Provider
          name: anthropic
          spec:
            type: anthropic
            apiKey: ${GOCLAW_TOKEN}
        - kind: Agent
          name: research-agent
          spec:
            provider: anthropic
            model: claude-3-5-sonnet-20241022
            instructions: |
              You are a research assistant...
```

### Update GCPlane Manifest Without Redeploying

```bash
# Edit the ConfigMap directly
kubectl edit configmap goclaw-gcplane-manifest -n goclaw-prod

# GCPlane will detect the change and reconcile automatically
# Check logs
kubectl logs -n goclaw-prod -l app.kubernetes.io/component=gcplane -f
```

### Monitor Drift Notifications

If webhook configured, GCPlane sends Slack messages when:
- Actual state differs from desired state (drift detected)
- Reconciliation completes successfully
- Errors occur during reconciliation

Example Slack message:
```
🔔 GCPlane Drift Notification
Namespace: goclaw-prod
Resource: Provider/anthropic
Status: OUT_OF_SYNC
Expected: apiKey=sk-xxx...
Actual: apiKey=sk-yyy...
Action: Auto-reconciling...
```

---

## Upgrading the Chart

### Check Current Release

```bash
helm list -n goclaw-prod

# Get deployed version
helm get values goclaw-prod -n goclaw-prod
```

### Update Chart Version

1. **Pull Latest Chart**:
```bash
git pull origin main
# Or if using Helm repo:
helm repo update
```

2. **Review Changelog**:
```bash
cat charts/goclaw/Chart.yaml | grep version
```

3. **Test Upgrade (Dry-Run)**:
```bash
helm upgrade goclaw-prod ./charts/goclaw \
  -f values-production.yaml \
  -n goclaw-prod \
  --dry-run \
  --debug
```

4. **Perform Upgrade**:
```bash
helm upgrade goclaw-prod ./charts/goclaw \
  -f values-production.yaml \
  -n goclaw-prod \
  --wait \
  --timeout=10m
```

**Zero-Downtime Rollout**:
- RollingUpdate strategy ensures old pods kept running until new pods ready
- PDB minAvailable ensures at least 1 pod always available
- Liveness/readiness probes ensure new pods healthy before routing traffic

### Rollback if Issues

```bash
helm rollback goclaw-prod -n goclaw-prod

# Or specific revision
helm history goclaw-prod -n goclaw-prod
helm rollback goclaw-prod 1 -n goclaw-prod
```

---

## Troubleshooting

### Pod Not Starting

**Symptoms**: Pod stuck in `Pending`, `Init:Error`, or `CrashLoopBackOff`

**Diagnosis**:
```bash
kubectl describe pod -n goclaw <pod-name>
kubectl logs -n goclaw <pod-name> --previous
```

**Common Issues**:

1. **Database Not Ready** (Init Container Failure):
   ```bash
   # Check if DB pod is running
   kubectl get pods -n goclaw -l app.kubernetes.io/component=db
   
   # If external DB, verify connectivity
   kubectl exec -it -n goclaw <server-pod> -- \
     pg_isready -h <db-host> -p 5432
   ```

2. **Image Pull Failed**:
   ```bash
   # Check image pull secret
   kubectl get secret -n goclaw <pull-secret>
   
   # Verify registry credentials
   docker login ghcr.io
   ```

3. **Insufficient Resources**:
   ```bash
   kubectl top nodes
   kubectl describe node <node-name>
   
   # Scale down or add nodes
   kubectl scale deployment goclaw -n goclaw --replicas=1
   ```

### Pod Running but Health Check Failing

**Symptoms**: Pod in `Running` state but readiness probe failing

**Diagnosis**:
```bash
# Check probe configuration
kubectl get pod -n goclaw <pod-name> -o yaml | grep -A20 readinessProbe

# Test health endpoint manually
kubectl exec -it -n goclaw <pod-name> -- \
  curl http://localhost:18790/health
```

**Solutions**:
- Increase initialDelaySeconds if startup slow
- Check application logs: `kubectl logs -n goclaw <pod-name>`
- Verify config/secrets mounted correctly

### Database Connection Issues

**Symptoms**: Server pod logs show `connection refused` or `password authentication failed`

**Diagnosis**:
```bash
# Check database credentials
kubectl get secret -n goclaw my-goclaw -o yaml

# Verify DSN construction (check env vars)
kubectl exec -it -n goclaw <server-pod> -- env | grep POSTGRES

# Test database connectivity
kubectl exec -it -n goclaw <db-pod> -- psql -U goclaw -d goclaw
```

**Solutions**:
- Verify externalDatabase settings match actual database
- Ensure database user/password correct
- Check database network accessibility (firewall, security groups)

### Ingress Not Routing Traffic

**Symptoms**: HTTP 404 or 503 when accessing via ingress

**Diagnosis**:
```bash
# Check ingress configuration
kubectl get ingress -n goclaw
kubectl describe ingress -n goclaw my-goclaw

# Check backend service endpoints
kubectl get endpoints -n goclaw my-goclaw

# Test service connectivity directly
kubectl port-forward svc/my-goclaw 18790:18790
curl http://localhost:18790/health
```

**Solutions**:
- Verify service selector labels match pod labels
- Check ingress annotations for controller-specific settings
- Ensure TLS certificate valid (if using HTTPS)

### PVC Stuck in Pending

**Symptoms**: PVC not provisioned; `kubectl get pvc` shows `Pending`

**Diagnosis**:
```bash
# Check storage class
kubectl get storageclass

# Describe PVC for events
kubectl describe pvc -n goclaw <pvc-name>

# Check volume provisioner logs
kubectl logs -n kube-system -l app=local-path-provisioner
```

**Solutions**:
- Ensure storage class exists: `kubectl get storageclass`
- Provision storage manually or use dynamic provisioner
- Check node disk space: `kubectl top nodes`

### GCPlane Not Reconciling

**Symptoms**: GCPlane pod running but manifest not applied

**Diagnosis**:
```bash
# Check GCPlane logs
kubectl logs -n goclaw <gcplane-pod> -f

# Verify token injected
kubectl exec -it -n goclaw <gcplane-pod> -- env | grep GOCLAW_TOKEN

# Check manifest ConfigMap
kubectl get configmap -n goclaw goclaw-gcplane-manifest -o yaml
```

**Solutions**:
- Ensure secret with GOCLAW_TOKEN exists
- Verify manifest YAML is valid: `kubectl apply --dry-run=client -f manifest.yaml`
- Check webhook URL is reachable (if configured)

---

## Performance Tuning

### Server CPU/Memory

**Increase if**:
- High request throughput (> 100 req/s)
- Large agent state or memory usage

```yaml
server:
  resources:
    limits:
      cpu: "4"           # 4 CPUs
      memory: 4Gi        # 4 GB
```

### PostgreSQL Performance

**Increase if**:
- Slow query times
- High connection count (> 100 connections)

```yaml
db:
  resources:
    limits:
      cpu: "2"
      memory: 4Gi
```

**Additional tuning** (external DB):
- Enable query logging: `log_min_duration_statement = 100`
- Increase shared_buffers: `shared_buffers = 8GB` (25% of RAM)
- Increase work_mem: `work_mem = 64MB` per connection

### Browser Sidecar Concurrency

**Increase if**:
- High web automation request volume
- Chrome OOM kills

```yaml
browser:
  shmSize: 4Gi          # Larger shared memory
  resources:
    limits:
      cpu: "4"
      memory: 4Gi
```

---

## Backup & Recovery

### Backup Procedure

**PostgreSQL**:
```bash
# Sidecar database
kubectl exec -n goclaw <db-pod> -- \
  pg_dump -U goclaw goclaw > backup.sql

# External database
pg_dump -h <db-host> -U goclaw goclaw > backup.sql
```

**PVCs** (via storage snapshots):
```bash
# AWS EBS snapshot
aws ec2 create-snapshot --volume-id <vol-id> \
  --description "goclaw-data-backup"

# GCP persistent disk snapshot
gcloud compute disks snapshot <disk-name> \
  --snapshot-names=goclaw-data-backup
```

**Secrets**:
```bash
kubectl get secret -n goclaw -o yaml > secrets-backup.yaml
# Store securely (vault, AWS Secrets Manager, etc.)
```

### Restore Procedure

**PostgreSQL**:
```bash
kubectl exec -i -n goclaw <db-pod> -- \
  psql -U goclaw goclaw < backup.sql
```

**PVCs**:
- Restore from snapshot via cloud provider console
- Or recreate from backup and re-upload data

**Secrets**:
```bash
kubectl apply -f secrets-backup.yaml
```

---

## Monitoring & Logging

### Key Metrics

```bash
# CPU/Memory usage
kubectl top pods -n goclaw

# Pod restart count (indicates instability)
kubectl get pods -n goclaw -o custom-columns=\
NAME:.metadata.name,\
RESTARTS:.status.containerStatuses[0].restartCount

# Events (errors, warnings)
kubectl get events -n goclaw --sort-by='.lastTimestamp'
```

### Log Aggregation

**Collect Logs**:
```bash
# All pods
kubectl logs -n goclaw -l app.kubernetes.io/name=goclaw --all-containers=true

# Stream logs from all server pods
kubectl logs -n goclaw -l app.kubernetes.io/component=server -f --all-containers=true
```

**Export to External System** (Elasticsearch, Datadog, etc.):
```bash
# Deploy Fluent Bit or similar log forwarder
helm install fluent-bit fluent/fluent-bit \
  -f values-fluent-bit.yaml \
  -n logging
```

---

## Security Best Practices

1. **Use Managed PostgreSQL** (production)
   - AWS RDS, Google Cloud SQL, Azure Database
   - Automated backups, encryption at rest, encrypted in-transit

2. **Network Policies**
   ```yaml
   networkPolicy:
     enabled: true
   ```
   - Restrict ingress/egress to only required services
   - Deny pod-to-pod by default

3. **Image Scanning**
   ```bash
   # Scan images for vulnerabilities
   trivy image ghcr.io/nextlevelbuilder/goclaw:v2.65.0-full
   ```

4. **Secret Rotation**
   - Rotate `GOCLAW_GATEWAY_TOKEN` regularly
   - Use external secret manager (Vault, AWS Secrets Manager)

5. **RBAC**
   - Limit cluster access to CI/CD pipelines
   - Use Workload Identity for service-to-service auth

6. **Audit Logging**
   - Enable Kubernetes audit logging
   - Monitor API calls for suspicious activity


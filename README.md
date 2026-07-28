# Netdisco Helm Chart

[![Release](https://img.shields.io/github/v/release/netdisco/helm-charts?label=chart&sort=semver)](https://github.com/netdisco/helm-charts/releases)
[![Validate Helm Chart](https://github.com/netdisco/helm-charts/actions/workflows/chart-validate.yaml/badge.svg)](https://github.com/netdisco/helm-charts/actions/workflows/chart-validate.yaml)
[![Release Helm Chart](https://github.com/netdisco/helm-charts/actions/workflows/helm-release.yaml/badge.svg)](https://github.com/netdisco/helm-charts/actions/workflows/helm-release.yaml)
[![License](https://img.shields.io/github/license/netdisco/helm-charts)](LICENSE)

Helm chart for deploying [Netdisco](https://netdisco.org) on Kubernetes or Red Hat OpenShift (RHOS).

The chart lives under `charts/netdisco/`.

## Install from the chart repository

```bash
helm repo add netdisco https://netdisco.github.io/helm-charts
helm repo update

# Quick try-out with the bundled PostgreSQL
helm install netdisco netdisco/netdisco \
  --set postgresql.enabled=true

# OpenShift / external PostgreSQL
helm install netdisco netdisco/netdisco \
  --set db.host=mypostgres.example.com \
  --set db.password=secret \
  --set route.host=netdisco.apps.mycluster.example.com
```

Show the chart's full default values with:

```bash
helm show values netdisco/netdisco
```

## Prerequisites

- Helm 3
- A PostgreSQL database (external recommended for production)
- For RHOS: the Vault Agent Injector and/or External Secrets Operator if using credential injection

The sections below use the chart from a local checkout (clone this repo first). For production deployments use the chart repository commands above instead.

## Quick start — local cluster (bundled PostgreSQL)

```bash
kind create cluster
helm install netdisco ./charts/netdisco -f charts/netdisco/values-test.yaml
```

## OpenShift deployment (external PostgreSQL)

```bash
helm install netdisco ./charts/netdisco \
  -f charts/netdisco/values-openshift.yaml \
  --set db.host=mypostgres.example.com \
  --set db.password=secret \
  --set route.host=netdisco.apps.mycluster.example.com
```

## With Vault Agent (DB) + ESO (SNMP credentials)

```bash
helm install netdisco ./charts/netdisco \
  -f charts/netdisco/values-openshift.yaml \
  -f charts/netdisco/values-vault-eso.yaml \
  --set vault.dbPath=secret/data/netdisco/db \
  --set eso.vaultPath=secret/data/netdisco/snmp
```

Edit `values-vault-eso.yaml` to set your Vault KV paths and adjust the `deviceAuthTemplate` to match your SNMP credential structure.

By default Vault Agent injects only the DB password and `db.host`/`db.user`/`db.name` are taken from values. If your Vault binding returns the full connection payload (host/port/database/user/password as JSON), set `vault.fullCredentials=true` to render the entire `database:` block from Vault and skip the static fields:

```bash
helm install netdisco ./charts/netdisco \
  -f charts/netdisco/values-openshift.yaml \
  -f charts/netdisco/values-vault-eso.yaml \
  --set vault.fullCredentials=true \
  --set vault.dbPath=secret/data/netdisco/db
```

Override `vault.dbKeys.{host,port,name,user,password}` if your payload uses different JSON key names (defaults: `hostname`, `port`, `database_name`, `username`, `password`).

## Linting and dry-run (no cluster needed)

```bash
# Lint
docker run --rm -v $(pwd)/charts/netdisco:/chart alpine/helm:latest lint /chart

# Render templates
docker run --rm -v $(pwd)/charts/netdisco:/chart alpine/helm:latest template netdisco /chart \
  -f /chart/values-openshift.yaml --set db.password=secret
```

## Values files

| File | Purpose |
|---|---|
| `values.yaml` | Full reference with all defaults and comments |
| `values-test.yaml` | Local testing with bundled PostgreSQL |
| `values-openshift.yaml` | RHOS production (external DB, Route, arbitrary UID) |
| `values-vault-eso.yaml` | Credential injection overlay (Vault Agent + ESO) |

## Credential injection architecture

When `vault.enabled` or `eso.enabled` is set, an init container merges the non-sensitive ConfigMap with the injected credential files before the application starts:

```
ConfigMap (deployment.yml) ──┐
                              ├─ init container ─► emptyDir ─► app
Vault/ESO Secret (creds) ────┘
```

Vault Agent injects DB credentials; ESO syncs SNMP `device_auth` from Vault KV into a k8s Secret.

## Scheduler / worker split for horizontal scaling

By default the backend Deployment runs all three roles in one process: Scheduler (submits jobs on cron), Manager (pulls from PostgreSQL queue), and Poller (executes discovery jobs). Running multiple replicas is unsafe in this mode because every replica's Scheduler submits duplicate jobs every minute.

Enable `scheduler.enabled=true` to split them:

- A dedicated **scheduler** Deployment (1 replica, no pollers) is created
- The **backend** Deployment receives `NETDISCO_NO_SCHEDULER=1` and can now safely scale to N replicas

```yaml
scheduler:
  enabled: true
  replicas: 1
  resources:
    limits:
      cpu: 200m
      memory: 1Gi

backend:
  replicas: 2   # safe to scale now
  resources:
    limits:
      cpu: "2"
      memory: 4Gi
```

Requires `NETDISCO_NO_SCHEDULER` and `NETDISCO_WORKERS_TASKS` env var support in the netdisco backend binary (available from the `feature-combined` image tag onwards).

### CPU / memory autoscaling (built-in HPA)

Enable the built-in `autoscaling/v2` HPA to scale the backend on CPU and memory:

```yaml
backend:
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 80       # % of requested CPU
    targetMemoryUtilization: 80    # % of requested memory
    scaleDownStabilizationSeconds: 300  # avoid killing pods mid-job
```

When `hpa.enabled=true` the `replicas` field is omitted from the Deployment so the HPA has sole ownership of the replica count.

### Autoscaling with KEDA (queue-depth)

Netdisco exposes `netdisco_jobs{status="queued"}` on the web pod's `/metrics` endpoint. If [KEDA](https://keda.sh) is installed and Prometheus is scraping the web pod, you can drive autoscaling directly from queue depth — this scales more precisely than CPU/memory because it reacts to actual work rather than resource pressure.

Replicas are calculated as `ceil(queueDepth / threshold)`. Tune the threshold to match `workers.tasks` and your expected queue depth — a full discovery run can queue hundreds of jobs, so a value around 50 gives a gradual ramp without immediately pegging at maxReplicas:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: netdisco-backend
spec:
  scaleTargetRef:
    name: netdisco-backend
  minReplicaCount: 1
  maxReplicaCount: 4
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: netdisco_queued_jobs
        query: netdisco_jobs{status="queued",tenant="netdisco"}
        threshold: "50"   # ceil(queueDepth/threshold) replicas — 200 jobs → 4 replicas
```

## Argo CD note: db-init hook timing

The db-init Job defaults to `helm.sh/hook: post-install,pre-upgrade` — a fresh
install waits until the chart's own ServiceAccount/ConfigMap/Secret exist,
while every later upgrade migrates the schema before new pods roll out. This
relies on Helm CLI's real distinction between install and upgrade.

Argo CD maps both `pre-install` and `pre-upgrade` onto the same `PreSync`
phase and can't tell an install from an upgrade — every operation is a
"sync". So under Argo CD this hook combination also fires as `PreSync` on a
fresh sync, before those resources exist, and can fail the same way 0.5.0
did. If you deploy via Argo CD, override the annotation to force single,
always-safe `PostSync` timing instead (an explicit Argo CD hook annotation
takes priority over `helm.sh/hook`):

```yaml
dbInit:
  annotations:
    argocd.argoproj.io/hook: PostSync
```

## See also

- [netdisco/netdisco](https://github.com/netdisco/netdisco) — the main application
- [netdisco/netdisco-docker](https://github.com/netdisco/netdisco-docker) — container images used by this chart

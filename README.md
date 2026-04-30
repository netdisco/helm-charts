# Netdisco Helm Chart

Helm chart for deploying [Netdisco](https://netdisco.org) on Kubernetes or Red Hat OpenShift (RHOS).

The chart lives under `charts/netdisco/`.

## Prerequisites

- Helm 3
- A PostgreSQL database (external recommended for production)
- For RHOS: the Vault Agent Injector and/or External Secrets Operator if using credential injection

No local Helm installation is required — all commands below use the official `alpine/helm` Docker image.

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

## See also

- [netdisco/netdisco](https://github.com/netdisco/netdisco) — the main application
- [netdisco/netdisco-docker](https://github.com/netdisco/netdisco-docker) — container images used by this chart

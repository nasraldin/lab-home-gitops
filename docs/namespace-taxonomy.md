# Kubernetes namespace taxonomy

Few purpose-grouped namespaces for the Dev Homelab cluster. Prefer these over
one-namespace-per-app.

## Canonical namespaces

| Namespace       | Contents                                                                 |
| --------------- | ------------------------------------------------------------------------ |
| `ai-tools`      | n8n, LibreChat, LiteLLM, OpenClaw (k8s)                                  |
| `observability` | Grafana, Prometheus, Loki, Tempo, Alertmanager, OTel Collector           |
| `database`      | CNPG Postgres + Redis / RabbitMQ / MariaDB **operators**                 |
| `artifacts`     | Harbor, Verdaccio                                                        |
| `storage`       | Longhorn (+ StorageClass extras)                                         |
| `security`      | cert-manager, Kyverno, external-secrets, Infisical operator + auth hint  |
| `gitops`        | GitLab Runner (k8s), KEDA                                                |
| `apps`          | Keycloak / SonarQube (until Docker/VM cutover) + future microservices    |

## Left alone (system / bootstrap)

| Namespace                         | Why                                                         |
| --------------------------------- | ----------------------------------------------------------- |
| `kube-system`, `kube-public`      | Kubernetes system                                           |
| `default`                         | Leave empty                                                 |
| `cilium-*` / CNI namespaces       | Networking                                                  |
| **`argocd`**                      | **Deviation:** Argo CD control plane + Application CRs stay |

### Why `argocd` is not renamed to `gitops`

Bootstrap (`lab-home-k8s` `install-argocd.sh`) and every Application
`metadata.namespace` assume `argocd`. Renaming the control plane requires a
reinstall and breaks in-flight sync. Logical “gitops” workloads (runner, KEDA)
live in `gitops`; Argo itself remains in `argocd`.

### GitLab Runner → `gitops`

Runner is CI infrastructure that syncs/builds for GitOps, not a product app.
Placing it in `gitops` (with KEDA) keeps namespace count low without a thin `ci`
namespace.

### Keycloak / SonarQube → `apps`

Not in the core taxonomy. Placement docs move them to Docker/VM; while they
remain in-cluster they share `apps` rather than dedicated namespaces.

## Old → new mapping

| Old namespace                 | New namespace   |
| ---------------------------- | --------------- |
| `litellm`, `n8n`, `librechat`, `ai` | `ai-tools` |
| `observability`              | `observability` |
| `data`, `data-system`, `cnpg-system`, `redis-operator`, `rabbitmq-operator`, `mariadb-operator` | `database` |
| `harbor`, `verdaccio`        | `artifacts`     |
| `longhorn-system`            | `storage`       |
| `cert-manager`, `kyverno`, `external-secrets`, `infisical-operator-system` | `security` |
| `keda`, `gitlab-runner`      | `gitops`        |
| `keycloak`, `sonarqube`      | `apps`          |
| `argocd`                     | `argocd` (keep) |

## PVC / data cutover

Namespace moves do **not** migrate PersistentVolumeClaims. Lab default is
**recreate** (acceptable for testing):

1. Sync GitOps with new destinations (`CreateNamespace=true`).
2. Confirm new pods Healthy in target namespaces.
3. Delete empty old namespaces (`kubectl delete ns <old>`).
4. Re-seed day-0 secrets via `lab-home-k8s/scripts/apply-bootstrap-secrets.sh`
   if Infisical is not yet syncing.

For stateful apps you care about (Harbor registry, LibreChat Mongo, CNPG):
export/backup before prune, or keep the old namespace until data is copied.
Longhorn volumes stay on disk until the PVC is deleted — pruning an old
namespace deletes its PVCs when reclaim policy allows.

## Cross-namespace DNS

| Service                         | FQDN                                              |
| ------------------------------- | ------------------------------------------------- |
| LiteLLM                         | `http://litellm.ai-tools.svc.cluster.local:4000/v1` |
| CNPG Postgres (RW)              | `postgres-rw.database.svc.cluster.local`          |
| OTel Collector                  | `http://otel-collector.observability.svc:4318`    |
| Infisical universal-auth Secret | `security` / `infisical-universal-auth`           |

## Cutover checklist (when API is up)

```bash
# After pushing GitOps changes and Argo has synced:
kubectl get ns
kubectl get applications -n argocd -o custom-columns=NAME:.metadata.name,DEST:.spec.destination.namespace,SYNC:.status.sync.status,HEALTH:.status.health.status

# Spot-check
kubectl -n ai-tools get pods
kubectl -n observability get pods
kubectl -n database get pods
kubectl -n artifacts get pods
kubectl -n storage get pods
kubectl -n security get pods
kubectl -n gitops get pods
kubectl -n apps get pods

# Remove empty leftovers (only when empty / data abandoned)
for ns in litellm n8n librechat ai data data-system cnpg-system redis-operator \
  rabbitmq-operator mariadb-operator harbor verdaccio longhorn-system \
  cert-manager kyverno external-secrets infisical-operator-system keda \
  gitlab-runner keycloak sonarqube; do
  kubectl get ns "$ns" >/dev/null 2>&1 || continue
  kubectl delete ns "$ns" --wait=false
done
```

## Cutover status (2026-07-30)

Purpose namespaces are **live** and hold the workloads (LB Services in
`ai-tools`, `artifacts`, `observability`, `storage`, `apps`, `argocd`, …).

| Item | Status |
| ---- | ------ |
| Canonical NS present | `ai-tools`, `observability`, `database`, `artifacts`, `storage`, `security`, `gitops`, `apps`, `argocd` |
| Empty legacy NS | Safe to delete when empty: `harbor`, `keycloak`, `sonarqube`, `gitlab-runner`, `cert-manager`, `kyverno`, `external-secrets`, `infisical-operator-system`, `keda`, `data-system`, … |
| Operator leftovers | `longhorn-system` / `cnpg-system` may still exist from charts — confirm empty or migrated before prune |
| PVC/data | Lab recreate for most apps on NS move |
| Cross-NS DNS | `litellm.ai-tools`, `postgres-rw.database`, Infisical UA Secret in `security` |
| Durable GitOps | Push `lab-home-gitops` to GitLab LAN + GitHub so Argo matches tree |

Kept: `kube-*`, `default`, `cilium-secrets`, `argocd`.

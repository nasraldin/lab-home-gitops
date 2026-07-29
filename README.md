# lab-home-gitops

Argo CD **app-of-apps** for the **Dev Homelab** Kubernetes cluster. Bootstrap
Cilium and Argo once from [`lab-home-k8s`](https://github.com/nasraldin/lab-home-k8s);
everything else syncs from this repository.

**Documentation:** https://nasraldin.github.io/dev-homelab/

## Layout

| Path               | Purpose                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| `bootstrap/`       | Root Application → `clusters/single`                                    |
| `clusters/single/` | Platform Application lists                                              |
| `platform/`        | Namespaces, security, storage, database, artifacts, observability, …    |
| `apps/`            | Workloads (AI tools + app-of-apps)                                      |
| `docs/`            | [Namespace taxonomy](docs/namespace-taxonomy.md)                        |

## Namespaces (few, purpose-grouped)

| Namespace       | Contents                                      |
| --------------- | --------------------------------------------- |
| `ai-tools`      | n8n, LibreChat, LiteLLM, OpenClaw             |
| `observability` | Prometheus, Grafana, Loki, Tempo, OTel        |
| `database`      | CNPG + Redis/RabbitMQ/MariaDB operators       |
| `artifacts`     | Harbor, Verdaccio                             |
| `storage`       | Longhorn                                      |
| `security`      | cert-manager, Kyverno, ESO, Infisical operator|
| `gitops`        | GitLab Runner, KEDA                           |
| `apps`          | Keycloak/Sonar (interim) + future services    |
| `argocd`        | Argo CD control plane (kept; see taxonomy doc)|

Full mapping and PVC cutover: [docs/namespace-taxonomy.md](docs/namespace-taxonomy.md).

## Sync order (waves)

1. Canonical namespaces (`platform/namespaces`)
2. cert-manager → `security`
3. metrics-server → `kube-system`
4. external-secrets + Infisical operator → `security`
5. Kyverno + policies + Infisical bootstrap hint → `security`
6. KEDA → `gitops`
7. Longhorn → `storage`
8. Data operators + CNPG cluster → `database`
9. Keycloak, SonarQube → `apps` (InfisicalSecret CRs in-path)
10. Harbor, Verdaccio → `artifacts`
11. Observability → `observability`
12. GitLab Runner → `gitops`
13. AI apps → `ai-tools`

OTLP endpoint (after sync): `http://otel-collector.observability.svc:4318`
(LAN LB `.110`). See [OpenTelemetry](https://nasraldin.github.io/dev-homelab/architecture/opentelemetry).

LiteLLM in-cluster: `http://litellm.ai-tools.svc.cluster.local:4000/v1`

Secrets: [Infisical](https://nasraldin.github.io/dev-homelab/architecture/secrets-and-infisical) —
seed from `lab-home-k8s` ansible, sync via InfisicalSecret CRs.
Auth Secret lives in `security` (`infisical-universal-auth`).

Supply chain: [docs](https://nasraldin.github.io/dev-homelab/architecture/supply-chain) —
Kyverno Audit first; Cosign Enforce after CI signs Harbor images.

**Not included yet:** Falco, Wazuh (see [Wazuh placement](https://nasraldin.github.io/dev-homelab/architecture/wazuh)).

Public URLs are configured in `lab-home-k8s` group_vars — see the
[Dev Homelab docs](https://nasraldin.github.io/dev-homelab/) (daily guide and
[public URLs](https://nasraldin.github.io/dev-homelab/access/public-urls)).

## CI

`.gitlab-ci.yml` includes lint/validate jobs from
[`pipeline-templates`](https://github.com/nasraldin/pipeline-templates).
Path-scoped pipelines run only for changed `platform/<component>/` directories.

GitLab is the canonical CI host for this repo; Argo CD syncs from GitLab
(`homelab/lab-home-gitops` on `gitlab.nasraldin.com`).

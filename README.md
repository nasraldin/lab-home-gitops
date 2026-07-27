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
| `platform/`        | cert-manager, Longhorn, Kyverno, secrets bootstrap, Keycloak, Harbor, … |
| `apps/`            | Your workloads                                                          |

## Sync order (waves)

1. cert-manager
2. metrics-server
3. external-secrets + Infisical operator
4. Kyverno + Audit policies + Infisical bootstrap hint
5. keda
6. longhorn
7. data operators (CNPG, Redis, RabbitMQ, MariaDB)
8. keycloak, sonarqube (InfisicalSecret CRs in-path)
9. harbor (Trivy on), verdaccio
10. observability (Prometheus, Grafana, Loki, Tempo, **OTel Collector**)
11. gitlab-runner + KEDA

OTLP endpoint (after sync): `http://otel-collector.observability.svc:4318`
(LAN LB `.110`). See [OpenTelemetry](https://nasraldin.github.io/dev-homelab/architecture/opentelemetry).

Secrets: [Infisical](https://nasraldin.github.io/dev-homelab/architecture/secrets-and-infisical) —
seed from `lab-home-k8s` ansible, sync via InfisicalSecret CRs.

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

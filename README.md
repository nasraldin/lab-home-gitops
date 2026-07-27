# lab-home-gitops

Argo CD **app-of-apps** for the **Dev Homelab** Kubernetes cluster. Bootstrap
Cilium and Argo once from [`lab-home-k8s`](https://github.com/nasraldin/lab-home-k8s);
everything else syncs from this repository.

**Documentation:** https://nasraldin.github.io/dev-homelab/

## Layout

| Path | Purpose |
| ---- | ------- |
| `bootstrap/` | Root Application → `clusters/single` |
| `clusters/single/` | Platform Application lists |
| `platform/` | cert-manager, Longhorn, data operators, Keycloak, SonarQube, Harbor, … |
| `apps/` | Your workloads |

## Sync order (waves)

1. cert-manager  
2. metrics-server  
3. external-secrets + Infisical operator  
4. keda  
5. longhorn  
6. data operators (CNPG, Redis, RabbitMQ, MariaDB)  
7. keycloak, sonarqube  
8. harbor, verdaccio  
9. observability (Prometheus, Grafana, Loki, Tempo)  
10. gitlab-runner + KEDA  

**Not included** (practice lab only): Istio, Kyverno, affinity-demo.

Public URLs are configured in `lab-home-k8s` group_vars — see the
[Dev Homelab docs](https://nasraldin.github.io/dev-homelab/) (daily guide and
[public URLs](https://nasraldin.github.io/dev-homelab/access/public-urls)).

## CI

`.gitlab-ci.yml` includes lint/validate jobs from
[`pipeline-templates`](https://github.com/nasraldin/pipeline-templates).
Path-scoped pipelines run only for changed `platform/<component>/` directories.

GitLab is the canonical CI host for this repo; Argo CD syncs from GitLab
(`homelab/lab-home-gitops` on `gitlab.nasraldin.com`).

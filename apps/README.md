# AI client apps (GitOps)

All chat/RAG/workflow UIs talk to **LiteLLM** (OpenAI-compatible gateway).
LiteLLM is the only service that calls Ollama on `llm-01`.

```text
[LibreChat | n8n | OpenClaw]
            │
            ▼
     LiteLLM :4000  (.108)
            │
            ▼
 Ollama on llm-01 (.26 :11434)
 models: gemma4:12b · gemma4:12b-think · qwen3.5:9b
```

| App       | URL / LB                         | Notes                         |
| --------- | -------------------------------- | ----------------------------- |
| LiteLLM   | http://litellm.lab (.108:4000)   | Gateway                       |
| LibreChat | http://chat.lab                  | Admin seeded; registration off |
| OpenClaw  | http://openclaw.lab (.113:18789) | Agent gateway (`ai-tools`); NPM auto `#token=` bootstrap — see `docs/operations/openclaw.md` |
| n8n       | http://n8n.lab                   | Workflows                     |

**Namespace:** all AI client apps deploy to Kubernetes namespace **`ai-tools`**.
LiteLLM DNS: `http://litellm.ai-tools.svc.cluster.local:4000/v1`.
See [docs/namespace-taxonomy.md](../docs/namespace-taxonomy.md).

Models (LibreChat dropdown): `gemma4:12b` (fast), `gemma4:12b-think`, `qwen3.5:9b`.

**GPU path:** privileged LXC `llm-01` with `/dev/dri` + `/dev/kfd` (host `amdgpu`, **no** VFIO).
See [ollama-llm-01](https://github.com/nasraldin/homelab/blob/main/docs/operations/ollama-llm-01.md).
`ai-01` is standby until decommission (do not delete until `ollama ps` shows GPU).

**Prerequisites**

1. Host `amdgpu` loaded; VFIO conf disabled (`lab-home-k8s/scripts/host-igpu-for-lxc.sh` + reboot)
2. `llm-01` running with Ollama on `0.0.0.0:11434`
3. `curl http://192.168.68.26:11434/api/tags` shows `gemma4:12b`
4. LiteLLM synced and healthy (`curl http://192.168.68.108:4000/health/readiness`)
5. Cluster has Longhorn + Cilium LB pool (`.100–.119`)

Full write-up: [ai-stack](https://nasraldin.github.io/dev-homelab/architecture/ai-stack).

**Security**

Helm values include lab keys for LiteLLM / LibreChat / n8n. Rotate them for any
shared or long-lived environment. Prefer ExternalSecrets later.

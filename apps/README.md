# AI client apps (GitOps)

All chat/RAG/workflow UIs talk to **LiteLLM** (OpenAI-compatible gateway).
LiteLLM is the only service that calls Ollama on `ai-01`.

```text
[LibreChat | n8n | OpenClaw]
            │
            ▼
     LiteLLM :4000  (.108)
            │
            ▼
 Ollama on ai-01 :11434
 models: gemma4:12b · gemma4:12b-think · qwen3.5:9b
```

| App       | URL / LB                         | Notes                         |
| --------- | -------------------------------- | ----------------------------- |
| LiteLLM   | http://litellm.lab (.108:4000)   | Gateway                       |
| LibreChat | http://chat.lab                  | Model dropdown                |
| OpenClaw  | http://openclaw.lab (.113:18789) | Agent gateway (`ai` namespace) |
| n8n       | http://n8n.lab                   | Workflows                     |

**Namespace plan:** new AI workloads go in Kubernetes namespace **`ai`** (OpenClaw first).
Existing apps (`librechat`, `litellm`, …) stay in their own namespaces until a planned
migration — consolidating them is a cutover, not a rename.

Models (LibreChat dropdown): `gemma4:12b` (fast), `gemma4:12b-think`, `qwen3.5:9b`.

**Hugepages on ai-01:** keep enabled (`hugepages: 2`). Proxmox memory % stays near 0% by
design; use `free -h` / `ollama ps` inside the guest for real RAM use.

**Prerequisites**

1. Host VFIO: `1002:150e` bound to `vfio-pci` ([gpu-passthrough](https://nasraldin.github.io/dev-homelab/architecture/gpu-passthrough))
2. `ai-01` running (hugepages `"2"`) with Ollama on `0.0.0.0:11434`
3. `curl http://192.168.68.20:11434/api/tags` shows `gemma4:12b`
4. LiteLLM synced and healthy (`curl http://192.168.68.108:4000/health/readiness`)
5. Cluster has Longhorn + Cilium LB pool (`.100–.119`)

Full write-up: [ai-stack](https://nasraldin.github.io/dev-homelab/architecture/ai-stack).

**Security**

Helm values include lab keys for LiteLLM / LibreChat / n8n. Rotate them for any
shared or long-lived environment. Prefer ExternalSecrets later.

# AI client apps (GitOps)

All chat/RAG/workflow UIs talk to **LiteLLM** (OpenAI-compatible gateway).
LiteLLM is the only service that calls Ollama on `ai-01`.

```text
[LibreChat | AnythingLLM | n8n | Open WebUI]
                    │
                    ▼
         LiteLLM :4000  (.108)
                    │
                    ▼
     Ollama on ai-01 :11434  (gemma4:12b)
     · 8c / 16 GiB / 150 GiB · 890M VFIO · 2 MiB hugepages
```

| App         | LB IP          | Upstream                       |
| ----------- | -------------- | ------------------------------ |
| LiteLLM     | 192.168.68.108 | → `http://192.168.68.20:11434` |
| LibreChat   | 192.168.68.105 | → LiteLLM in-cluster           |
| AnythingLLM | 192.168.68.106 | → LiteLLM in-cluster           |
| n8n         | 192.168.68.107 | → LiteLLM (OpenAI cred in UI)  |
| Open WebUI  | 192.168.68.109 | → LiteLLM in-cluster           |

In-cluster base URL: `http://litellm.litellm.svc.cluster.local:4000/v1`  
LAN base URL: `http://192.168.68.108:4000/v1`  
API key: LiteLLM `masterkey` (see `litellm/apps.yaml`)  
Model: `gemma4:12b`

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

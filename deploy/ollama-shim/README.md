# Ollama-shim

Tiny stdlib-only HTTP service that exposes an Ollama-compatible API
(`/api/chat`, `/api/generate`, `/api/embed`, `/api/tags`, `/api/pull`,
`/api/show`, `/api/version`) and translates requests to the cluster's
OpenAI-compatible LiteLLM gateway plus the Ollama embeddings backend.

Lets existing Ollama clients — UIP's `alert-enricher`, IDE plugins,
ad-hoc scripts — point at this endpoint with **only an env-var change**.

## Endpoint

- External: `http://aicompute01.cnco1.tucows.cloud:31434` (or any compute node's
  IP on port 31434; NodePort routes cluster-wide)
- In-cluster: `http://domains-ollama-shim.lab-domains-sre.svc.cluster.local:11434`

The port (`11434`) and NodePort (`31434`) match the default Ollama port,
so callers configured for a remote Ollama need no port change either.

## What works (and what doesn't)

| Endpoint | Behavior |
|---|---|
| `GET /` | Liveness OK |
| `GET /api/tags` | Returns the three available models: `qwen3-32b-thinking`, `qwen3-235b-thinking`, `bge-m3` |
| `GET /api/version` | Returns `{"version":"shim-1.0"}` |
| `POST /api/chat` | Translated to `/v1/chat/completions` on gateway. Maps `options.num_predict` -> `max_tokens`, `format:"json"` -> JSON response, `think:true/false` -> chat-template `enable_thinking` |
| `POST /api/generate` | Same as `/api/chat` but returns Ollama's `response` field instead of `message.content` |
| `POST /api/embed`, `/api/embeddings` | Forwarded verbatim to the dedicated `embeddings-ollama` pod which already speaks native Ollama API |
| `POST /api/pull` | No-op success — models are pre-loaded by vLLM |
| `POST /api/show` | Returns a minimal stub; real metadata is in the gateway |
| Streaming responses | **Not supported.** All chat requests are forwarded non-streaming (`stream:false`). Clients that depend on `stream:true` won't work. |

## UIP integration changes

Two env vars on the UIP host (`/home/fash/uip/.env`):

```bash
# alert-enricher
OLLAMA_URL=http://aicompute01.cnco1.tucows.cloud:31434
OLLAMA_MODEL=qwen3-32b-thinking

# opensrs-health-api still uses OpenAI shape - point at the gateway directly
OPENAI_API_BASE=http://aicompute01.cnco1.tucows.cloud:31440/v1
OPENAI_MODEL=qwen3-32b-thinking
```

After updating, restart UIP services:

```bash
cd /home/fash/uip
docker compose up -d alert-enricher opensrs-health-api
```

The in-stack `ollama` container in `docker-compose.yml` can be retired
(scale to 0 replicas or remove the service block entirely) once UIP is
talking to the cluster.

## Apply

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' apply -k 'C:\Users\fash\Desktop\AI Nodes\deploy\ollama-shim'
```

## Scale up

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre scale deploy/ollama-shim --replicas=1
```

## Sanity check (from outside the cluster)

```bash
# list models
curl http://aicompute01.cnco1.tucows.cloud:31434/api/tags

# chat completion (matches enricher.py's request shape)
curl http://aicompute01.cnco1.tucows.cloud:31434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-32b-thinking","messages":[{"role":"user","content":"Say ready"}],"stream":false,"think":false,"options":{"num_predict":40}}'
```

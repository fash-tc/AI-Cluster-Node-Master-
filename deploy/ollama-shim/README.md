# Ollama-shim

Tiny stdlib-only HTTP service that exposes an Ollama-compatible API
(`/api/chat`, `/api/generate`, `/api/embed`, `/api/tags`, `/api/pull`,
`/api/show`, `/api/version`) and translates requests to the cluster's
OpenAI-compatible LiteLLM gateway plus the Ollama embeddings backend.

Lets existing Ollama clients — IDE plugins, agent frameworks, ad-hoc
scripts, anything hard-coded to the Ollama API — point at this endpoint
with **only an env-var change**.

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

## Pointing a client at the shim

Typical Ollama client configurations expose an `OLLAMA_URL` /
`OLLAMA_HOST` / `--host` flag. Set it to the shim's endpoint:

```bash
OLLAMA_URL=http://aicompute01.cnco1.tucows.cloud:31434
OLLAMA_MODEL=qwen3-32b-thinking   # or qwen3-235b-thinking, bge-m3
```

The client should "just work" without further changes. If you want to
verify outside any framework, the sanity-check command below speaks
the exact wire format Ollama clients use.

## Apply

```bash
kubectl apply -k deploy/ollama-shim
```

## Scale up

```bash
kubectl -n lab-domains-sre scale deploy/ollama-shim --replicas=1
```

## Sanity check (from outside the cluster)

```bash
# list models
curl http://aicompute01.cnco1.tucows.cloud:31434/api/tags

# chat completion
curl http://aicompute01.cnco1.tucows.cloud:31434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-32b-thinking","messages":[{"role":"user","content":"Say ready"}],"stream":false,"think":false,"options":{"num_predict":40}}'

# embed text
curl http://aicompute01.cnco1.tucows.cloud:31434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"bge-m3","input":"hello world"}'
```

# ollama-shim

A stdlib-only Python HTTP service (~250 lines) that exposes an
Ollama-compatible API in front of the OpenAI gateway. Lets existing
Ollama clients connect to this cluster with **only an env-var change**.

## What it is

- **Image:** `python:3.12-slim` (no pip installs needed — pure stdlib)
- **Code:** [`deploy/ollama-shim/configmap.yaml`](../../deploy/ollama-shim/configmap.yaml)
  contains the full Python source mounted into the pod
- **Bundle:** [`deploy/ollama-shim/`](../../deploy/ollama-shim/)
- **Resources:** 100m-1 CPU, 64-256 MiB RAM

## Where it runs

Single-replica deployment with `RollingUpdate` strategy. Stateless — can
be scaled to N replicas behind the same NodePort if traffic warrants
it. Currently lands on whichever node is least loaded CPU-side.

## What it translates

| Ollama endpoint | Translated to | Notes |
|---|---|---|
| `POST /api/chat` | `POST /v1/chat/completions` on gateway | Maps `options.num_predict` → `max_tokens`, `format:"json"` → `response_format`, `think:bool` → `chat_template_kwargs.enable_thinking`, `tools` passes through |
| `POST /api/generate` | `POST /v1/chat/completions` (wrapped as single-user-message) | Returns Ollama's `response` field instead of `message.content` |
| `POST /api/embed`, `/api/embeddings` | Forwarded verbatim to `embeddings-ollama:11434` | No translation — pgvector backend is already native Ollama |
| `GET /api/tags` | Static list of available models | Hard-coded in `TAGS` constant in shim source |
| `POST /api/pull` | No-op `{"status":"success"}` | Models are pre-loaded by vLLM |
| `POST /api/show` | Minimal stub response | Most clients ignore this; full fidelity not needed |
| `GET /api/version` | `{"version":"shim-1.0"}` | Compatibility shim |
| `GET /`, `HEAD /` | 200 OK | Liveness checks |

## Endpoints

- External: `http://aicompute01.cnco1.tucows.cloud:31434` — drop-in
  replacement for `http://localhost:11434` in any Ollama client config
- In-cluster: `http://domains-ollama-shim.lab-domains-sre.svc.cluster.local:11434`

## Response coalescing — the `reasoning_content` fix

vLLM's reasoning parser sometimes puts the final answer in
`message.reasoning_content` instead of `message.content` (see
[gotchas](../gotchas.md#vllm-puts-answers-in-reasoning_content)). The
shim handles this by reading both fields and using whichever is
populated:

```python
content = msg.get("content") or msg.get("reasoning_content") or ""
```

OpenAI clients still see the original split; the shim only patches
this on the Ollama path. So Ollama clients see clean `message.content`,
and OpenAI clients can choose to coalesce themselves.

## What the shim does NOT do

- **Streaming.** All requests are forwarded with `stream:false` to the
  gateway. The shim returns a single complete JSON response. Clients
  that send `"stream": true` get a non-streaming response anyway.
- **Per-model auth.** No API key enforcement.
- **Rate limiting.** None.

## Adding a new model to the tags list

When the gateway gains a new logical model (e.g. you added it to
`litellm-gateway-config`), you also need to add it to the shim's
`TAGS` list so `/api/tags`-aware clients can discover it:

```python
# in deploy/ollama-shim/configmap.yaml, find TAGS = [...]
{
    "name": "my-new-model",
    "model": "my-new-model",
    "size": 0,
    "digest": "n/a",
    "details": {
        "family": "qwen3",
        "parameter_size": "32B",
        "quantization_level": "FP8",
    },
},
```

Then re-apply and roll the shim:

```bash
kubectl apply -k deploy/ollama-shim
kubectl -n lab-domains-sre rollout restart deploy/ollama-shim
```

## When to send traffic through the shim vs the gateway

| Use the shim | Use the gateway directly |
|---|---|
| You have an Ollama-pinned client that you can't or don't want to change | Anything new that you control |
| You need `/api/embed` (the shim is also the only external entry point for embeddings) | You want streaming responses |
| You want a single URL that serves both chat and embeddings | You want to call vLLM features the shim doesn't translate (logit bias, etc.) |

The shim has an inherent latency cost (~1–2 ms per call) since it
proxies through Python. Negligible for typical chat — measurable only
if you're issuing thousands of QPS.

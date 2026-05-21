# litellm-gateway

The single OpenAI-compatible front door for the cluster's LLMs. A thin
LiteLLM Proxy that knows about the model backends, load-balances the
32B HA pair, and gives us request logging.

## What it is

- **Image:** `ghcr.io/berriai/litellm:main-latest`
- **Role:** OpenAI Proxy. Receives `/v1/chat/completions` requests,
  picks a backend by `model:` field, forwards, and returns the response.
- **Bundle:** [`deploy/litellm-gateway/`](../../deploy/litellm-gateway/)
- **Resources:** 1-4 CPU, 1-4 GiB RAM. No GPU.

## Where it runs

`litellm-gateway` is a single-replica deployment with `RollingUpdate`
strategy and `maxSurge: 1, maxUnavailable: 0`. It lands wherever the
scheduler places it (no `nodeSelector` constraint) — typically
`aicompute01` since it's the least loaded CPU-side.

## Routing config

The `litellm-gateway-config` ConfigMap defines two logical models, each
mapped to one or two vLLM backends:

| Logical model | Backend(s) | Strategy |
|---|---|---|
| `qwen3-235b-thinking` | `domains-qwen3-235b:8000` | Single replica |
| `qwen3-32b-thinking` | `domains-qwen3-32b-aicompute02:8000`, `domains-qwen3-32b-aicompute03:8000` | `simple-shuffle` (random pick per request) |

Router settings:

- `num_retries: 2` — automatic retry on backend errors
- `request_timeout: 600` — 10-min ceiling per request (enough for the
  235B on long-context inputs)
- `cooldown_time: 30` — after a failure, that backend is skipped for
  30 s before being tried again

## Endpoints

- External: `http://aicompute01.cnco1.tucows.cloud:31440/v1`
- In-cluster: `http://domains-llm-gateway.lab-domains-sre.svc.cluster.local:4000`

Both speak standard OpenAI: `/v1/chat/completions`, `/v1/completions`,
`/v1/models`, `/health/readiness`, `/health/liveliness`.

## Adding a new model

Edit `configmap.yaml` and add an entry under `model_list`:

```yaml
- model_name: my-new-model
  litellm_params:
    model: hosted_vllm/my-new-model
    api_base: http://domains-my-new-model.lab-domains-sre.svc.cluster.local:8000/v1
    api_key: not-needed
```

Then:

```bash
kubectl apply -k deploy/litellm-gateway
kubectl -n lab-domains-sre rollout restart deploy/litellm-gateway
```

The gateway restarts in ~10 seconds.

To make the same model available via the Ollama shim, also add it to
the static `TAGS` list in `deploy/ollama-shim/configmap.yaml` and
restart the shim.

## What the gateway does NOT do

- **Authentication.** No API key enforcement, no rate limiting per
  caller. Every consumer with network access gets unlimited use.
- **Streaming back-pressure handling.** Streaming requests are passed
  through to vLLM verbatim.
- **Model warm-up checks.** `/health/readiness` returns 200 the moment
  the proxy is listening — even if no backend is yet up. Don't rely on
  it for "is the model ready" — query the model's own `/v1/models`
  endpoint instead.

## Logs

```bash
kubectl -n lab-domains-sre logs -f deploy/litellm-gateway
```

Logs are JSON. Each `/health/readiness` heartbeat appears as a
`"GET /health/readiness HTTP/1.1" 200`. Real chat completions log with
the model name and timing.

## What happens during a 32B replica outage

LiteLLM marks the failing backend "in cooldown" for 30 s after a
network or 5xx error. During cooldown, that backend is skipped — all
traffic goes to the surviving replica. After cooldown, it's retried
opportunistically.

This means a single 32B replica restart causes at most ~30 s of
elevated latency for the `qwen3-32b-thinking` route, not an outage.
Restart both replicas at once and the model is unavailable until at
least one comes back.

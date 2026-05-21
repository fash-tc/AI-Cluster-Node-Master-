# LiteLLM Gateway

Single OpenAI-compatible front door for the cluster's reasoning models.
Routes by `model` name to the right vLLM backend, load-balances the 32B
HA pair, and gives us per-request logging — useful as an audit trail
for any caller that needs to retrace what was sent and what came back.

## Exposed model names

| `model` value sent by client | Backend |
|---|---|
| `qwen3-235b-thinking` | aicompute01 → `domains-qwen3-235b:8000` |
| `qwen3-32b-thinking` | round-robin across aicompute02 and aicompute03 |

## Endpoint

- Base URL: `http://aicompute01.cnco1.tucows.cloud:31440/v1`
  (or any compute node — same port, NodePort routes cluster-wide)
- API key: any placeholder if a client requires one

## Example call

```bash
curl http://aicompute01.cnco1.tucows.cloud:31440/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-32b-thinking",
    "messages": [{"role":"user","content":"What is the cluster status?"}],
    "chat_template_kwargs": {"enable_thinking": false}
  }'
```

Switch `model` to `qwen3-235b-thinking` for deep reasoning, or set
`enable_thinking: true` to get `<think>` blocks back.

## Apply

```bash
kubectl apply -k deploy/litellm-gateway
```

## Scale up (after at least one backend is Ready)

```bash
kubectl -n lab-domains-sre scale deploy/litellm-gateway --replicas=1
```

## Update routes

Edit `configmap.yaml`, `apply -k`, then bounce the gateway:

```bash
kubectl -n lab-domains-sre rollout restart deploy/litellm-gateway
```

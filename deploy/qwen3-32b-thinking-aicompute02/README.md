# Qwen3-32B (hybrid thinking) on aicompute02

Fast tier reasoning replica A. Same weights as `qwen3-32b-thinking-aicompute03`
form an HA pair behind the LiteLLM gateway's `qwen3-32b-thinking` route.

## Topology

- Pinned to `aicompute02.cnco1.tucows.cloud`.
- 4 GPU shares reserved (whole physical GH200).
- FP8 weights ~32 GiB on HBM, leaves room for 32K context and 16 concurrent
  sequences.

## Endpoint

- Direct: `http://aicompute02.cnco1.tucows.cloud:31442/v1`
- Through gateway: same endpoint as 03, gateway round-robins.

## Hybrid thinking

This is the same model on both nodes. To toggle thinking mode per request,
pass `chat_template_kwargs={"enable_thinking": true|false}` in your OpenAI
chat completion call. Default is `true` for the `Thinking` variants. For
snappy classification or simple Q&A, set it to false.

## Apply

```bash
kubectl apply -k deploy/qwen3-32b-thinking-aicompute02
```

## Scale up

```bash
kubectl -n lab-domains-sre scale deploy/qwen3-32b-thinking-aicompute02 --replicas=1
```

Expect 10–25 min first start (model pull).

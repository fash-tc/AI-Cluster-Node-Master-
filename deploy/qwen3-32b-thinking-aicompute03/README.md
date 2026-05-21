# Qwen3-32B (hybrid thinking) on aicompute03

Fast tier reasoning replica B. Pair of `qwen3-32b-thinking-aicompute02`.

## Endpoint

- Direct: `http://aicompute03.cnco1.tucows.cloud:31443/v1`
- Through gateway: same as 02, gateway round-robins.

## Apply

```bash
kubectl apply -k deploy/qwen3-32b-thinking-aicompute03
```

## Scale up

```bash
kubectl -n lab-domains-sre scale deploy/qwen3-32b-thinking-aicompute03 --replicas=1
```

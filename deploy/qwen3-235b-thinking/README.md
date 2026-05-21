# Qwen3-235B-A22B-Thinking on aicompute01

Single-node MoE reasoning model. Heavy reasoning route for long-context
analysis, multi-step problem decomposition, and any prompt that benefits
from explicit chain-of-thought.

## Topology

- Pinned to `aicompute01.cnco1.tucows.cloud` (one physical GH200).
- 4 GPU shares reserved (the whole physical GPU — no time-slice sharing).
- Uses `--cpu-offload-gb 144` so the 235B FP8 weights (~220 GB) ride a mix
  of HBM3 (96 GB) and Grace LPDDR5X. NVLink-C2C carries cold experts at
  ~450 GB/s, which is much better than traditional CPU offload over PCIe.
- No cross-node networking — sidesteps the NCCL-socket-fallback issue.

## Endpoint

- Direct (bypass gateway): `http://aicompute01.cnco1.tucows.cloud:31441/v1`
- Through gateway: `http://aicompute01.cnco1.tucows.cloud:31440/v1` with
  `model: qwen3-235b-thinking`
- API key: any placeholder if a client requires one

## Safety defaults

- Starts at `replicas: 0`. Manual scale-up only.
- Conservative initial limits: `max-model-len 65536`, `max-num-seqs 2`,
  `max-num-batched-tokens 8192`.
- `--kv-cache-dtype fp8` to leave HBM for activations and big context.
- `--reasoning-parser deepseek_r1` strips/exposes `<think>` blocks correctly.

## Apply paused manifests

```bash
kubectl apply -k deploy/qwen3-235b-thinking
```

## Scale up (after node health check)

```bash
kubectl -n lab-domains-sre scale deploy/qwen3-235b-thinking --replicas=1
```

Expect 30–90 min first start while the model downloads to the PVC.

## Wait for ready

```bash
kubectl -n lab-domains-sre rollout status deploy/qwen3-235b-thinking --timeout=120m
```

## Rollback

```bash
kubectl -n lab-domains-sre scale deploy/qwen3-235b-thinking --replicas=0
```

## If aicompute01 goes NotReady during first scale-up

Do not retry. Pause everything, pull node events, dmesg, and GPU health.
The 480B Ollama deployment hit this once and the lesson stands.

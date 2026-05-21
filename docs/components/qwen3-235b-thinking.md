# qwen3-235b-thinking

The heavyweight reasoning model. Single-node MoE on `aicompute01` with
Grace-memory CPU offload.

## What it is

- **Model:** `Qwen/Qwen3-235B-A22B-Thinking-2507-FP8`
- **Architecture:** Mixture-of-experts, 235 B total parameters, 22 B
  active per token, 128 experts with 8 routed per token
- **Precision:** FP8 (E4M3) for weights, FP8 for KV cache
- **Context:** 65 536 tokens (model supports 256 K natively; we limit
  for conservative memory budget)
- **Concurrency:** 2 sequences simultaneously
- **Bundle:** [`deploy/qwen3-235b-thinking/`](../../deploy/qwen3-235b-thinking/)

## Where it runs

Pinned to `aicompute01.cnco1.tucows.cloud` via `nodeSelector`. Reserves
all 4 GPU shares (the whole physical GH200). No co-tenants on the GPU.

## How the GH200 unified memory helps here

A naive load of 235B FP8 weights (~220 GiB) does not fit in HBM3
(96 GiB). vLLM's `--cpu-offload-gb 144` tells the engine to keep the
hot path on HBM3 and overflow into Grace LPDDR5X over NVLink-C2C
(~450 GB/s). MoE models are unusually well-suited to this — only 8 of
128 experts fire per token, so most weights stay cold and live happily
on the (large but slower) Grace side.

In practice: weights split roughly 96 GiB on HBM3 + ~140 GiB on Grace.
Inference TTFT is acceptable; TPS is meaningfully lower than the 32B
but still usable for non-interactive paths.

## vLLM startup args (from configmap + deployment)

```
vllm serve Qwen/Qwen3-235B-A22B-Thinking-2507-FP8 \
  --served-model-name qwen3-235b-thinking \
  --host 0.0.0.0 --port 8000 \
  --trust-remote-code \
  --enable-expert-parallel \
  --kv-cache-dtype fp8 \
  --cpu-offload-gb 144 \
  --max-model-len 65536 \
  --max-num-seqs 2 \
  --max-num-batched-tokens 8192 \
  --reasoning-parser deepseek_r1 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --disable-uvicorn-access-log
```

## Endpoints

| URL | API | Use case |
|---|---|---|
| `http://aicompute01.cnco1.tucows.cloud:31441/v1` | OpenAI direct | Bypass the gateway. Only for debugging or pinning. |
| `http://aicompute01.cnco1.tucows.cloud:31440/v1` (gateway, `model: qwen3-235b-thinking`) | OpenAI via LiteLLM | Default path. |
| `http://aicompute01.cnco1.tucows.cloud:31434` (Ollama shim, `model: qwen3-235b-thinking`) | Ollama API | For Ollama-pinned clients. |

The gateway routes `qwen3-235b-thinking` to the single 235B replica;
there is no load balancing because there is only one replica.

## When to use it

- Multi-step reasoning that benefits from explicit chain-of-thought
- Long-context document analysis (paste tens of KB of text and ask for a
  structured summary or decomposition)
- Synthesis across multiple sources ("given these N pieces of evidence,
  what conclusion do they jointly support?")
- Code review / refactoring proposals on non-trivial code
- Backstop when the 32B's answer is shallow or low-confidence

Avoid for:

- High-QPS, latency-sensitive paths — the 235B's queue saturates at 2
  concurrent sequences and TPS is ~4× slower than the 32B
- Anything where a 32B answer is good enough — escalate intentionally,
  not by default

## Start-up time

- Cold start from `replicas: 0` with cached weights: ~3 min
- Cold start with no PVC cache (first ever or after PVC delete): 30-60
  min (Hugging Face pull of ~220 GiB)

## Scaling notes

This deployment is intentionally single-replica. Adding a second 235B
replica would need a second physical GH200 node and would still not
have cross-node speedup (no RDMA). If 235B throughput becomes a
problem, the right move is **route easy requests to the 32B and reserve
the 235B for the genuinely hard ones**, not "more 235B replicas."

## History

- The cluster previously hosted Ollama-based `qwen3-coder:480b` on
  `aicompute03`. A first smoke test crashed the node to `NotReady`,
  which is why every new deployment now starts paused-by-default.
- A vLLM/Ray `deepseek-v4-flash` with pipeline-parallel-size=3 was
  configured but never validated because of the no-RDMA constraint.
- This 235B deployment is the first big model on this cluster that
  actually stays up in real use.

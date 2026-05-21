# qwen3-32b-thinking

The fast tier. Two single-node FP8 replicas, one on `aicompute02` and
one on `aicompute03`. Load-balanced behind the LiteLLM gateway.

## What it is

- **Model:** `Qwen/Qwen3-32B-FP8`
- **Architecture:** Dense 32 B parameters with hybrid thinking (toggle
  per request)
- **Precision:** FP8 weights, default KV cache type
- **Context:** 32 768 tokens
- **Concurrency:** 16 sequences simultaneously per replica
- **Bundles:**
  - [`deploy/qwen3-32b-thinking-aicompute02/`](../../deploy/qwen3-32b-thinking-aicompute02/)
  - [`deploy/qwen3-32b-thinking-aicompute03/`](../../deploy/qwen3-32b-thinking-aicompute03/)

## Where it runs

Each replica is pinned to its node via `nodeSelector`. Both reserve all
4 GPU shares per their physical GH200. Two replicas → about 2× the
throughput of one, and HA against single-replica failure.

## vLLM startup args

```
vllm serve Qwen/Qwen3-32B-FP8 \
  --served-model-name qwen3-32b-thinking \
  --host 0.0.0.0 --port 8000 \
  --trust-remote-code \
  --max-model-len 32768 \
  --max-num-seqs 16 \
  --max-num-batched-tokens 32768 \
  --reasoning-parser deepseek_r1 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --disable-uvicorn-access-log
```

## Hybrid thinking

The model is a "thinking" variant that can run in two modes per
request:

- `enable_thinking: false` — produces a direct answer, no
  `<think>...</think>` reasoning trace. Use this for snappy
  classification, structured output, simple Q&A.
- `enable_thinking: true` (default) — produces a reasoning trace
  followed by the answer. Use this when you want chain-of-thought,
  multi-step reasoning, or self-correction.

Pass it via `chat_template_kwargs` in OpenAI-shaped requests:

```python
client.chat.completions.create(
    model="qwen3-32b-thinking",
    messages=[...],
    extra_body={"chat_template_kwargs": {"enable_thinking": False}},
)
```

Via the Ollama shim, use the top-level `"think": false` field instead:

```json
{"model":"qwen3-32b-thinking","messages":[...], "think": false}
```

## Endpoints

| URL | API | Backend |
|---|---|---|
| `http://aicompute01.cnco1.tucows.cloud:31440/v1` (gateway, `model: qwen3-32b-thinking`) | OpenAI | Load-balanced across both replicas |
| `http://aicompute02.cnco1.tucows.cloud:31442/v1` | OpenAI direct | aicompute02 replica only |
| `http://aicompute03.cnco1.tucows.cloud:31443/v1` | OpenAI direct | aicompute03 replica only |
| `http://aicompute01.cnco1.tucows.cloud:31434` (Ollama shim, `model: qwen3-32b-thinking`) | Ollama API | Load-balanced via gateway |

For nearly all use, hit the gateway. The direct ports are for
debugging or pinning to a specific replica during canaries.

## Why 32K context, not 131K

Qwen3-32B's native `max_position_embeddings` is 40 960. Going beyond
that requires YaRN rope scaling, which adds config surface and accuracy
risk. We picked **32K** as a clean round number well below the native
limit. For most use — a prompt plus a few thousand lines of retrieved
context — 32K is comfortable. See
[gotchas](../gotchas.md#qwen3-native-context-is-40k-not-131k) for the
details.

## Throughput sizing

Each replica handles 16 concurrent sequences. Two replicas → 32
concurrent. Above that, the gateway queues requests at the per-replica
vLLM layer and clients see latency rise.

If you anticipate sustained higher concurrency:

1. Lower per-request `max_tokens` (replies are usually overspec'd)
2. Bump `--max-num-seqs` (memory permitting; each new slot costs KV cache
   space)
3. Add a third replica on `aicompute01` — but only after retiring the
   235B, since both can't coexist on the same node's GPU

## When to use it

- Default tier for everything chat-shaped
- Classification, structured output (with `response_format: {"type": "json_object"}`)
- RAG-grounded Q&A (combine with rag-search retrieval)
- Conversational responses
- Tool calling via standard OpenAI `tools: [...]` interface

For deeper reasoning, see [qwen3-235b-thinking.md](qwen3-235b-thinking.md).

## Known interaction with the reasoning parser

vLLM's `--reasoning-parser deepseek_r1` extracts `<think>...</think>`
blocks into `message.reasoning_content`. When you set
`enable_thinking: false`, some vLLM versions still routed the answer
through the reasoning channel — leaving `message.content` empty and
the actual response in `message.reasoning_content`. The Ollama shim
handles this; OpenAI clients should read both fields (`content ??
reasoning_content`).

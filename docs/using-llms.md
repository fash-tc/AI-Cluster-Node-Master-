# Using the LLMs

The cluster exposes its language models through two API shapes — pick
whichever your client speaks.

## Pick your front door

| Use this | When |
|---|---|
| **OpenAI gateway** at `http://aicompute01.cnco1.tucows.cloud:31440/v1` | Default. Any tool that knows OpenAI (Python `openai` SDK, LangChain, LiteLLM clients, OpenAI-compatible IDE plugins). |
| **Ollama shim** at `http://aicompute01.cnco1.tucows.cloud:31434` | Tools that hard-code Ollama: Continue, Cline, Roo, OpenWebUI's Ollama backend, ad-hoc scripts that call `/api/chat`. |

Any compute node IP (`aicompute01`/`02`/`03`) works on either port —
NodePorts route across the cluster.

## Available models

| `model:` value | Type | Default context | Tier |
|---|---|---|---|
| `qwen3-32b-thinking` | Dense 32B FP8, hybrid thinking | 32K | Fast (HA pair, load-balanced) |
| `qwen3-235b-thinking` | MoE 235B / 22B active, FP8 | 65K | Heavy (single node, Grace-offload) |
| `bge-m3` | Embeddings (1024-dim) | n/a | Embeddings only — use `/api/embed`, not chat |

`/v1/models` (gateway) returns the live list. `/api/tags` (shim) returns
the same list in Ollama's shape.

## OpenAI API

### Python — `openai` package

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://aicompute01.cnco1.tucows.cloud:31440/v1",
    api_key="any-non-empty-string",  # gateway doesn't enforce auth
)

# Fast tier — disable thinking for snappy structured-output tasks
resp = client.chat.completions.create(
    model="qwen3-32b-thinking",
    messages=[
        {"role": "system", "content": "Reply only with valid JSON."},
        {"role": "user", "content": "Classify: 'API p95 latency 80ms->2400ms'"},
    ],
    extra_body={"chat_template_kwargs": {"enable_thinking": False}},
    response_format={"type": "json_object"},
    max_tokens=300,
)
print(resp.choices[0].message.content)
```

### Python — thinking mode for deeper reasoning

```python
resp = client.chat.completions.create(
    model="qwen3-235b-thinking",  # heavy tier
    messages=[{"role": "user", "content": "Three alerts: ... explain why they correlate."}],
    extra_body={"chat_template_kwargs": {"enable_thinking": True}},
    max_tokens=2000,
)
# Final answer goes in message.content. The model's chain-of-thought is
# either in message.reasoning_content (preferred) or wrapped in <think>
# tags inside content, depending on vLLM version.
print(resp.choices[0].message.content)
```

### curl

```bash
curl http://aicompute01.cnco1.tucows.cloud:31440/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-32b-thinking",
    "messages": [{"role":"user","content":"Hello"}],
    "chat_template_kwargs": {"enable_thinking": false},
    "max_tokens": 100
  }'
```

### Streaming

Append `"stream": true`. Standard OpenAI server-sent-events format.

```python
stream = client.chat.completions.create(
    model="qwen3-32b-thinking",
    messages=[{"role": "user", "content": "Count to 5"}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

## Ollama API (via the shim)

The shim translates Ollama-shaped calls into OpenAI calls behind the
scenes. From the client's perspective, it looks exactly like a remote
Ollama daemon — same paths, same response shapes.

### Chat

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-32b-thinking",
    "messages": [{"role":"user","content":"Hello"}],
    "stream": false,
    "think": false,
    "options": {"num_predict": 200, "temperature": 0.7}
  }'
```

`think: false` is the shim's way of expressing
`chat_template_kwargs.enable_thinking: false`. `options.num_predict` is
translated to OpenAI's `max_tokens`. `format: "json"` is translated to
`response_format: {"type": "json_object"}`.

### Listing models

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/tags
```

Returns a static list of the three models. The shim's `/api/pull` is a
no-op (200 OK) because vLLM pre-loads weights; old Ollama clients that
do "ensure model is pulled before use" still work without surprises.

### Embeddings

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"bge-m3","input":"this is a test"}'
```

Returns `{"model":"bge-m3","embeddings":[[...1024 floats...]]}`. Multiple
inputs at once: pass `"input": ["text 1", "text 2", ...]` and you get an
array of arrays back.

The shim forwards embed calls directly to `embeddings-ollama` without
any translation, so this is fast and matches Ollama's documented format
exactly.

### Streaming caveat for the shim

The shim **does not support streaming**. It always sends `stream:false`
to the gateway and returns a single complete JSON response. If your
client passes `"stream": true` on `/api/chat`, the shim still returns a
single non-streaming response.

If you need streaming, use the OpenAI gateway directly.

## When to use which model

### `qwen3-32b-thinking` (the default — use this first)

- Alert triage (severity classification, service identification)
- Runbook Q&A grounded in RAG retrieval
- Structured output (JSON-mode classifications)
- Conversational responses
- "What does this Zabbix alert mean?" type questions

Two replicas, load-balanced. 16 concurrent sequences supported. 32K
context — plenty for an alert + a few thousand lines of relevant log.

### `qwen3-235b-thinking` (when 32B isn't enough)

- Multi-alert correlation ("why do these three alerts fire together?")
- Long log dump analysis (paste 30 KB of syslog, ask for a summary)
- Multi-step reasoning ("trace through this incident timeline")
- Postmortem drafting
- Anything where the 32B's answer feels shallow or uncertain

One replica. 2 concurrent sequences. 65K context. Roughly 4× slower per
token than the 32B and uses significantly more memory thanks to the
expert pool living in Grace LPDDR5X.

A reasonable pattern: try the 32B first; if it says "I'm not sure" or
produces a low-confidence classification, retry on the 235B.

### `bge-m3`

Don't use it for chat — it's an embedding-only model. Use it via
`/api/embed` (Ollama shim) or directly through the rag-search service.

## Things to know

### `reasoning_content` field

vLLM's `--reasoning-parser deepseek_r1` is active on both Qwen models.
When the model emits any `<think>...</think>` content, it gets extracted
into `message.reasoning_content`. The final answer should be in
`message.content`.

**Quirk:** with `enable_thinking: false`, some vLLM versions still route
the entire response into `reasoning_content` and leave `content: null`.
The Ollama shim coalesces these for you (`content ?? reasoning_content`)
but OpenAI clients see the raw response. If you find `content` empty,
also read `reasoning_content`.

### Tool calling

Both Qwen models register `--tool-call-parser hermes`. You can pass
`tools: [...]` in OpenAI-style requests and the model produces standard
`tool_calls` arrays. The Ollama shim passes `tools` through unchanged.

### Rate limiting

There is none, today. The gateway and the vLLM servers will queue
requests up to `--max-num-seqs` (16 for the 32B, 2 for the 235B). Beyond
that, requests block on the queue. Heavy concurrent load will degrade
everyone's latency rather than reject requests outright.

### Authentication

There is none, today. Anyone with network access to a compute node IP
can hit any endpoint. This matches the cluster's internal-only
deployment posture. If a public-facing wrapper is ever added, do auth
there — don't try to bolt it onto the gateway layer-by-layer.

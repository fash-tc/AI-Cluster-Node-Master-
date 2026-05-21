# Architecture

This document is the **deep dive**. Read [README.md](README.md) first for
the cluster-level overview; come here when you need to understand request
flow, why each component exists, or how the layers fit together.

## The big picture

```
                              ┌───────────────────────────────────────┐
                              │  External clients (anywhere with      │
                              │  network access to a compute node):   │
                              │    - Any OpenAI-compatible app or SDK │
                              │    - Any Ollama-compatible client     │
                              │    - IDE plugins, agents, scripts     │
                              │    - This cluster's own jobs          │
                              └────────────────┬──────────────────────┘
                                               │
                       NodePorts on any of: aicompute01/02/03.cnco1.tucows.cloud
                                               │
   ┌──────────────────────┬─────────────────────┼────────────────────────┬────────────────────┐
   │                      │                     │                        │                    │
   ▼                      ▼                     ▼                        ▼                    ▼
:31434 (Ollama API)   :31440 (OpenAI)      :31441 OpenAI            :31442/:31443         :31445 (REST)
ollama-shim           litellm-gateway      qwen3-235b direct        32B direct (per node) rag-search
   │                      │                     │                        │                    │
   │  POST /api/chat       \\  routes by model     \\ no LB; one replica    \\ pin to replica     │
   │  POST /api/embed       \\                                              \\                   │
   │  POST /api/generate     \\                                              \\                  │
   │                          \\                                              \\                 │
   └─→ translates body ────────→  POST /v1/chat/completions ───────────────────→  vLLM         │
       to OpenAI shape          (LiteLLM picks backend                            (OpenAI       │
       (or forwards to           via model_list)                                   compatible   │
        embeddings for                                                             on :8000)    │
        /api/embed*)                                                                            │
                                                                                                │
                                                                              ┌─────────────────┘
                                                                              │
                                                                              ▼
                                                                  ┌─────────────────────┐
                                                                  │ rag-search service  │
                                                                  │ POST /v1/collections│
                                                                  │  /{name}/{op}       │
                                                                  └────┬────────────┬───┘
                                                                       │            │
                                                       embed via      │            │   query
                                                       Ollama API     │            │   pgvector
                                                                       ▼            ▼
                                                          ┌────────────────┐  ┌────────────┐
                                                          │ embeddings-    │  │ pgvector   │
                                                          │ ollama         │  │ (Postgres  │
                                                          │ (bge-m3 model) │  │  17 + vec) │
                                                          │ ClusterIP only │  │ ClusterIP  │
                                                          └────────────────┘  └────────────┘

                              ┌──── Physical GPU layout ────┐
                              │ aicompute01: Qwen3-235B     │
                              │ aicompute02: Qwen3-32B (A)  │
                              │ aicompute03: Qwen3-32B (B)  │
                              │ + CPU sidecars (pgvector,   │
                              │   bge-m3, gateway, shim,    │
                              │   rag-search, wiki-ingester)│
                              └─────────────────────────────┘
```

## Why each layer exists

### LiteLLM gateway (`:31440/v1`)

Acts as the **single OpenAI-compatible front door**. Two reasons it's
worth its own pod:

1. **Load balancing.** The two Qwen3-32B replicas appear as one logical
   `qwen3-32b-thinking` model. LiteLLM round-robins requests across them
   (configured via `router_settings.routing_strategy: simple-shuffle`).
2. **Audit trail.** Every chat completion passes through one log stream.
   Useful when the agent layer eventually needs request-level forensics.

LiteLLM also implements common OpenAI features the underlying vLLM
servers may handle inconsistently (retries, fallbacks, request timeout).

### Ollama shim (`:31434`)

A stdlib-only Python proxy (≈250 lines) that speaks the Ollama API and
translates to OpenAI behind the scenes. Lets clients hard-coded to Ollama
hit the cluster with **one env-var change** and no code changes.

Specifically translates:

- `POST /api/chat` → `POST /v1/chat/completions` on the gateway
- `POST /api/generate` → wraps as a single-message chat
- `POST /api/embed` / `/api/embeddings` → forwards to `embeddings-ollama`
- `GET /api/tags` → static list of available models
- `POST /api/pull` → no-op (models are pre-loaded)

Also coalesces vLLM's `message.reasoning_content` into `message.content`
so Ollama clients that only read `content` still see the answer.

**Not supported by the shim:** streaming responses. The shim forces
`stream: false` on every translation. If you need streaming, hit the
OpenAI gateway directly with `stream: true`.

### vLLM models (`:31441` / `:31442` / `:31443`)

Each Qwen model runs as a vLLM server speaking OpenAI on container port
8000, exposed as a NodePort.

- **Qwen3-235B-A22B-Thinking-2507-FP8** on `aicompute01`: MoE, 22B active
  per token, 235B total. Loaded with `--enable-expert-parallel
  --cpu-offload-gb 144 --kv-cache-dtype fp8`. The active experts live in
  HBM3 (96 GB); cold experts ride Grace LPDDR5X over NVLink-C2C
  (~450 GB/s). This is the only way to fit a 200B+ MoE on a single
  physical GPU.
- **Qwen3-32B FP8** on `aicompute02` and `aicompute03`: identical
  configuration, two replicas. Hybrid thinking — controlled per request
  via `chat_template_kwargs: {"enable_thinking": true/false}`. 32K
  context.

Both models register the `deepseek_r1` reasoning parser, which extracts
`<think>...</think>` blocks. (See [gotchas](docs/gotchas.md) for the
related `reasoning_content` quirk.)

### pgvector (`ClusterIP :5432`)

PostgreSQL 17 with the `vector` extension. The database is named `rag`,
and the user `rag` has create privileges. **Per-consumer tables** named
`rag_<collection>` give isolation: one team's writes never touch another
team's rows.

We use pgvector instead of a dedicated vector DB (Qdrant, Weaviate, etc.)
because jemalloc-bundled images crash on the GH200 64K-page kernel — a
real, reproducible blocker. PostgreSQL has none of that fragility on
aarch64-64K and is comfortable to tens of millions of vectors with HNSW.

### embeddings-ollama (`ClusterIP :11434`)

Ollama serving `bge-m3` (BAAI/bge-m3, 1024-dim multilingual, hybrid
dense+sparse-friendly embeddings). CPU-only — no GPU is taken because
embeddings are tiny and bursty. The pod uses ~1.2 GB RAM idle and bursts
under load.

Reachable directly (in-cluster) at `/api/embed`, or externally via the
Ollama shim's `:31434/api/embed`.

### rag-search (`:31445/v1/collections/...`)

A small Python service (≈300 lines) that wraps `bge-m3` + pgvector
behind a clean REST API. It exists so consumers don't have to:

- Ship a PostgreSQL client (psycopg) in their app
- Know the embedding dimension or chunk metadata schema
- Authenticate to pgvector

**Multi-tenancy by collection.** Each collection is a separate table
`rag_<name>` with the same shape: `id BIGINT PK`, `text TEXT`, `metadata
JSONB`, `embedding vector(1024)`, timestamps. Filter at search time via
`metadata @> {...}` (PostgreSQL JSONB containment).

### wiki-ingester (CronJob, daily 03:30 UTC)

A generic Confluence-to-RAG ingestion job. Configured via env vars
(`SPACE_KEY`, `COLLECTION`, `CONFLUENCE_SITE`) to pull a specific
Confluence space via Atlassian REST API, convert storage XHTML to text
via `html2text`, chunk by heading + paragraph, and upsert into a
rag-search collection. Idempotent — chunk IDs are deterministic
(`page_id * 1000 + chunk_index`) so re-running just refreshes content.

Today this CronJob is configured to sync one space into one collection;
to add a second space, deploy a second CronJob with different env vars
and a different secret.

## Request flows

### 1. Chat completion via OpenAI gateway

```
Client                LiteLLM (:31440)             vLLM 32B replica
  │                       │                              │
  │  POST /v1/chat/        │                              │
  │  completions           │                              │
  │  {model:               │                              │
  │   "qwen3-32b-thinking",│                              │
  │   messages: [...]      │                              │
  │   chat_template_kwargs:│                              │
  │    {enable_thinking:    │                              │
  │     false}}            │                              │
  ├──────────────────────► │                              │
  │                        │ pick backend                 │
  │                        │ (simple-shuffle)             │
  │                        ├────────────────────────────► │
  │                        │                              │  inference
  │                        │                              │  generate tokens
  │                        │ ◄────────────────────────────┤  return:
  │                        │   { choices:[                │   {choices:[{...
  │                        │     {message:{content?,      │     reasoning_content
  │                        │      reasoning_content?}}   │     or content}}]
  │                        │   ]}                          │
  │ ◄──────────────────────┤                              │
  │   200 OK               │                              │
  │   {choices:[...]}      │                              │
```

### 2. Chat completion via Ollama shim

```
Client                ollama-shim (:31434)        LiteLLM (:31440)
  │                       │                            │
  │  POST /api/chat        │                            │
  │  {model:               │                            │
  │   "qwen3-32b-thinking",│                            │
  │   messages: [...]      │                            │
  │   stream: false,       │                            │
  │   think: false,        │                            │
  │   options:{            │                            │
  │     num_predict:1024}} │                            │
  ├──────────────────────► │                            │
  │                        │ translate body:            │
  │                        │  num_predict→max_tokens    │
  │                        │  think→chat_template_kwargs│
  │                        │  format:"json"→            │
  │                        │   response_format          │
  │                        │ POST /v1/chat/completions  │
  │                        ├──────────────────────────► │
  │                        │                            │ (same as flow 1)
  │                        │ ◄──────────────────────────┤
  │                        │ translate response:        │
  │                        │  message.reasoning_content │
  │                        │  → message.content (if    │
  │                        │   content is null)         │
  │ ◄──────────────────────┤                            │
  │   200 OK               │                            │
  │   {message:{           │                            │
  │     content: "..."},   │                            │
  │    done: true}         │                            │
```

### 3. RAG search

```
Client                rag-search (:31445)         embeddings (CIP)        pgvector (CIP)
  │                       │                            │                       │
  │  POST                  │                            │                       │
  │  /v1/collections/      │                            │                       │
  │   my_collection/search │                            │                       │
  │  {query:"...",         │                            │                       │
  │   top_k: 10,           │                            │                       │
  │   filter:{             │                            │                       │
  │     category:"foo"}    │                            │                       │
  │  }                     │                            │                       │
  ├──────────────────────► │                            │                       │
  │                        │ POST /api/embed             │                       │
  │                        │ {model:"bge-m3",            │                       │
  │                        │  input: query}              │                       │
  │                        ├──────────────────────────► │                       │
  │                        │ ◄──────────────────────────┤                       │
  │                        │ {embeddings:[[...]]}        │                       │
  │                        │                             │                       │
  │                        │  SELECT id, text, metadata, │                       │
  │                        │  1 - (embedding <=>         │                       │
  │                        │       %s::vector) AS score  │                       │
  │                        │  FROM rag_my_collection     │                       │
  │                        │  WHERE metadata @> %s       │                       │
  │                        │  ORDER BY embedding <=>     │                       │
  │                        │       %s::vector            │                       │
  │                        │  LIMIT %s                   │                       │
  │                        ├─────────────────────────────────────────────────►   │
  │                        │ ◄─────────────────────────────────────────────────  │
  │                        │ [(id, text, metadata,       │                       │
  │                        │   score), ...]               │                       │
  │ ◄──────────────────────┤                             │                       │
  │   200 OK               │                             │                       │
  │   {results: [...]}     │                             │                       │
```

### 4. RAG upsert

```
Client                rag-search (:31445)         embeddings (CIP)        pgvector (CIP)
  │                       │                            │                       │
  │  POST                  │                            │                       │
  │  /v1/collections/      │                            │                       │
  │   {name}/upsert        │                            │                       │
  │  {id: 42,              │                            │                       │
  │   text: "...",         │                            │                       │
  │   metadata: {...}}     │                            │                       │
  ├──────────────────────► │                            │                       │
  │                        │ POST /api/embed             │                       │
  │                        ├──────────────────────────► │                       │
  │                        │ ◄──────────────────────────┤                       │
  │                        │ vec = [..1024 floats..]     │                       │
  │                        │                             │                       │
  │                        │ INSERT INTO rag_{name}      │                       │
  │                        │   (id, text, metadata,      │                       │
  │                        │    embedding)               │                       │
  │                        │ VALUES (...)                │                       │
  │                        │ ON CONFLICT (id) DO UPDATE  │                       │
  │                        ├─────────────────────────────────────────────────►   │
  │                        │ ◄─────────────────────────────────────────────────  │
  │ ◄──────────────────────┤                             │                       │
  │   200 OK {id:42,       │                             │                       │
  │     status:"ok"}        │                             │                       │
```

## Component dependencies

What needs what to be healthy:

```
ollama-shim       depends on  litellm-gateway, embeddings-ollama
litellm-gateway   depends on  qwen3-235b-thinking, qwen3-32b-thinking-aicompute02, qwen3-32b-thinking-aicompute03
rag-search        depends on  pgvector, embeddings-ollama
wiki-ingester     depends on  rag-search (which depends on pgvector + embeddings)
embeddings-ollama depends on  (nothing — leaf service)
pgvector          depends on  (nothing — leaf service)
qwen3-*           depends on  (nothing — leaf services on GPU)
```

The leaf services (vLLM models, pgvector, embeddings-ollama) can be
scaled or restarted independently. The middleware (gateway, shim,
rag-search) is stateless and can be rebooted at will — clients see a few
seconds of 502s during the rollover and then everything resumes.

## Multi-tenancy model

The cluster is **multi-tenant by convention, not by enforcement**:

- **Inference layer** (gateway, shim, vLLM models, embeddings) is shared.
  Every consumer hits the same models. No quotas, no per-tenant logging
  beyond LiteLLM's request log.
- **Data layer** (pgvector, rag-search collections) is per-team. Each
  team picks a collection name; their rows live in a separate table.
  Searches against one collection cannot surface rows from another.

Today there is one PostgreSQL role (`rag`) shared across collections.
That's fine while consumers are trusted. Once an external team comes on
board, the right tightening is: separate PG role + schema per consumer,
and a per-tenant Secret with their DSN. See
[docs/operations.md](docs/operations.md#hardening-pgvector-for-multi-tenant-use).

## History (what used to be here)

This cluster previously hosted three coding-focused models:

- `qwen3-coder-30b-vllm` (active for two weeks)
- `qwen3-coder-480b` (paused; a smoke test crashed `aicompute03` once)
- `ollama-qwen35` / `ollama-qwen35-02` (bare pods on `aicompute02`)

It also had a paused `deepseek-v4-flash-vllm` (3-node Ray/vLLM cluster
with pipeline-parallel=3) and a paused `minimax-m25-vllm`. Both were
experiments that ran into the no-RDMA + 64K-page constraints.

Everything above was retired on 2026-05-19 in favor of the current
single-node-per-model layout. The previous cluster state (full
`kubectl get all` dumps, PVCs, configmaps) is captured in a local
backup directory on the operator workstation; not committed here.

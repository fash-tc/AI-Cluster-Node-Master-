# AI Cluster — `prod-ai-k8` / `lab-domains-sre`

A self-hosted inference + retrieval platform running on three NVIDIA GH200
Grace-Hopper nodes. Exposes OpenAI-compatible and Ollama-compatible APIs for
LLM inference, a generic RAG service for any team's knowledge base, and an
embedding endpoint — all behind cluster-stable NodePorts.

This repository is the **living source of truth** for what is deployed,
how it is wired together, and how consumers integrate. Every manifest in
`deploy/` is what runs on the cluster today. Every doc in `docs/` reflects
the current state, not aspirations.

## What's in the cluster

| Service | Purpose | Front door |
|---|---|---|
| **Qwen3-235B-A22B-Thinking** | Heavyweight reasoning (long-context analysis, multi-step problem decomposition) | `:31441/v1` (OpenAI) |
| **Qwen3-32B (×2 replicas)** | Fast tier (classification, structured output, conversational, RAG-grounded Q&A) | `:31442/v1`, `:31443/v1` (OpenAI) |
| **LiteLLM Gateway** | OpenAI front door + load balancing across the 32B pair | `:31440/v1` |
| **Ollama Shim** | Translates Ollama API → OpenAI gateway (for Ollama-pinned clients) | `:31434` (Ollama API) |
| **rag-search** | Generic embeddings + pgvector wrapper; multi-tenant by collection | `:31445/v1/collections/{name}/...` |
| **pgvector** | PostgreSQL 17 + vector extension; in-cluster only | ClusterIP `:5432` |
| **Ollama embeddings (bge-m3)** | 1024-dim embeddings for RAG | ClusterIP `:11434/api/embed` (or via shim externally) |
| **wiki-ingester** | Daily CronJob syncing a configured Confluence space into a RAG collection | (internal job) |

All NodePorts are reachable on **any** compute node IP — `aicompute01`,
`aicompute02`, and `aicompute03` all route the same.

## Quick start

**Chat with a model (OpenAI client):**

```python
from openai import OpenAI
client = OpenAI(
    base_url="http://aicompute01.cnco1.tucows.cloud:31440/v1",
    api_key="anything",
)
resp = client.chat.completions.create(
    model="qwen3-32b-thinking",
    messages=[{"role": "user", "content": "Hello"}],
)
```

**Chat with a model (Ollama-pinned tool):**

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/chat \
  -d '{"model":"qwen3-32b-thinking","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

**Search a RAG collection:**

```bash
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/<your-collection>/search \
  -H "Content-Type: application/json" \
  -d '{"query":"what does this document say about X","top_k":3}'
```

**Embed text:**

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/embed \
  -d '{"model":"bge-m3","input":"text to embed"}'
```

## Repository layout

```
.
├── README.md               You are here
├── ARCHITECTURE.md         How the components fit together; request-flow diagrams
├── deploy/                 K8s manifests — what is actually running
│   ├── qwen3-235b-thinking/         Heavy reasoning model
│   ├── qwen3-32b-thinking-aicompute02/
│   ├── qwen3-32b-thinking-aicompute03/
│   ├── litellm-gateway/             OpenAI front door
│   ├── ollama-shim/                 Ollama-API translation layer
│   ├── rag-stack/                   pgvector + bge-m3 embeddings
│   ├── rag-search/                  REST wrapper over pgvector + embeddings
│   └── wiki-ingester/               Daily Confluence → RAG sync
└── docs/
    ├── hardware.md         GH200 specifics, why some images don't work
    ├── using-llms.md       How to call the LLMs (OpenAI + Ollama API)
    ├── using-rag.md        How to use the RAG service (upsert/search/delete)
    ├── operations.md       Deploy, scale, observe, recover
    ├── gotchas.md          Pitfalls discovered the hard way
    └── components/         Per-component deep dives
```

## Endpoints at a glance

All ports are NodePorts on any compute node. Substitute `aicompute01`,
`02`, or `03` interchangeably.

| Port | Service | API style | Use it for |
|---|---|---|---|
| `31440` | LiteLLM gateway | OpenAI `/v1/...` | Default chat path. Load-balances 32B; routes 235B to the single replica. |
| `31441` | Qwen3-235B direct | OpenAI `/v1/...` | Skip the gateway. Only if you need to pin the 235B replica directly. |
| `31442` | Qwen3-32B (aicompute02) | OpenAI `/v1/...` | Skip the gateway. Pin to a specific 32B replica. |
| `31443` | Qwen3-32B (aicompute03) | OpenAI `/v1/...` | Same, other replica. |
| `31434` | Ollama shim | Ollama `/api/chat`, `/api/embed`, `/api/tags`, `/api/generate` | Existing Ollama clients that won't speak OpenAI. |
| `31445` | rag-search | Custom REST | Vector search; embedding-backed retrieval. |

Internal-only (ClusterIP):

| DNS name | Service | Port |
|---|---|---|
| `domains-pgvector.lab-domains-sre.svc.cluster.local` | pgvector / PostgreSQL 17 | 5432 |
| `domains-embeddings.lab-domains-sre.svc.cluster.local` | Ollama (bge-m3) | 11434 |
| `domains-llm-gateway.lab-domains-sre.svc.cluster.local` | LiteLLM | 4000 |
| `domains-rag-search.lab-domains-sre.svc.cluster.local` | rag-search | 8081 |

## Models exposed

| Model name (use in API calls) | Backend | When to use |
|---|---|---|
| `qwen3-32b-thinking` | Qwen3-32B FP8, two replicas behind LiteLLM | Default. Classification, structured output (JSON-mode), conversational responses, RAG-grounded Q&A, tool calls. Hybrid thinking — pass `chat_template_kwargs: {"enable_thinking": false}` to disable the reasoning trace for snappier responses. |
| `qwen3-235b-thinking` | Qwen3-235B-A22B-Thinking-2507-FP8 (MoE) on one GH200 with Grace offload | Deep reasoning: long-context document analysis, multi-step problem decomposition, correlated-evidence synthesis. Slower and more expensive — escalate to it only when the 32B is uncertain. |
| `bge-m3` | Ollama serving BAAI/bge-m3 | 1024-dim embeddings. Multilingual, hybrid dense+sparse. Used internally by rag-search; can be called directly. |

## How to use the cluster

- **Calling LLMs** — see [docs/using-llms.md](docs/using-llms.md)
- **Building a RAG knowledge base** — see [docs/using-rag.md](docs/using-rag.md)
- **Operating the cluster** — see [docs/operations.md](docs/operations.md)
- **Architecture deep dive** — see [ARCHITECTURE.md](ARCHITECTURE.md)
- **Known pitfalls** — see [docs/gotchas.md](docs/gotchas.md)

## Hardware reality (read this once)

The cluster runs on three **NVIDIA GH200 Grace-Hopper Superchips**. Each
node has one physical GPU (96 GB HBM3) and ~600 GB Grace LPDDR5X CPU
memory bridged by NVLink-C2C at ~450 GB/s. Kubernetes exposes each
physical GPU as 4 time-sliced "shares".

Two non-obvious consequences shape every design decision in this repo:

1. **64K kernel pages.** GH200 Linux runs `6.8.0-1032-nvidia-64k`. Many
   container images bundle jemalloc compiled for 4K pages and crash
   immediately. Qdrant and the HuggingFace TEI server were both bitten by
   this; that's why we run **pgvector + Ollama-bge-m3** instead.
2. **No RDMA/GDR for cross-node traffic.** NCCL diagnostics showed pods
   fall back to TCP sockets. Multi-node tensor parallelism is slow and
   unreliable, so every model deployment is **single-node** with replicas
   for HA, not tensor-split across nodes.

[docs/hardware.md](docs/hardware.md) and [docs/gotchas.md](docs/gotchas.md)
have the full story.

## Living document policy

When you change what's deployed, change this repo too:

- New service → add `deploy/<name>/` + a doc under `docs/components/<name>.md`, link from this README.
- Changed config → update the manifest + the relevant doc; mention behavior changes.
- Removed service → delete the bundle + linked doc; note the removal in the relevant component doc's "history" section.

If `kubectl get all -n lab-domains-sre` diverges from `deploy/`, one of
them is wrong. Fix it.

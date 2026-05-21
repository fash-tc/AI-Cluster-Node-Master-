# embeddings-ollama

The embeddings backend. Ollama serving `bge-m3`, used by `rag-search`
for both upsert (embed-then-store) and search (embed-then-similarity)
paths. Also reachable directly via the Ollama shim for ad-hoc
embedding use.

## What it is

- **Image:** `ollama/ollama:latest`
- **Model:** `BAAI/bge-m3` (pulled at first start via `ollama pull bge-m3`)
- **Embedding dimension:** 1024
- **Bundle:** [`deploy/rag-stack/`](../../deploy/rag-stack/) (same bundle
  as pgvector)
- **Resources:** 4-8 CPU, 8-16 GiB RAM. **No GPU.**

## Why Ollama (not TEI or sentence-transformers)?

Three reasons:

1. **Already proven on this cluster.** Ollama ran for 17 days on this
   cluster pre-migration without issues. We trust its aarch64-64K
   compatibility.
2. **Multi-arch image.** The official `ollama/ollama` image has both
   `linux/amd64` and `linux/arm64` variants. Hugging Face's TEI server
   does not (see [gotchas](../gotchas.md#some-images-have-no-aarch64-build-at-all)).
3. **CPU is enough.** `bge-m3` runs comfortably on CPU at the throughput
   we need (a few embeddings per second per collection). Reserving a
   GPU share for this would be wasteful.

## Where it runs

Single-replica deployment, lands on whichever node has CPU capacity.
The model lives in a 20 GiB PVC (`embeddings-ollama-data`) so it
survives pod restarts. First start pulls ~1.2 GiB from Ollama's model
registry.

## Endpoints

| URL | Use case |
|---|---|
| `http://domains-embeddings.lab-domains-sre.svc.cluster.local:11434/api/embed` | In-cluster — used by `rag-search` |
| `http://aicompute01.cnco1.tucows.cloud:31434/api/embed` (via Ollama shim) | External — for ad-hoc embedding from outside the cluster |

The Ollama-shim just forwards `/api/embed` requests to this service
unchanged, so the request/response shapes are identical.

## Request shape

```bash
curl http://aicompute01.cnco1.tucows.cloud:31434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"bge-m3","input":"text to embed"}'
```

Response:

```json
{
  "model": "bge-m3",
  "embeddings": [[0.0210, 0.0064, -0.0377, ... 1021 more floats ...]]
}
```

Batch multiple inputs in one call:

```json
{"model": "bge-m3", "input": ["string 1", "string 2", "string 3"]}
```

Returns `embeddings: [[...], [...], [...]]`. Significantly faster
per-text than separate calls — use this when you have a queue of
documents to embed.

## Throughput

CPU-only `bge-m3` on the Grace CPU does roughly:

- Single short input (~50 tokens): ~50–100 ms
- Batch of 32 short inputs: ~300–500 ms total

These numbers are approximate; first call after a cold start is slower
because the model loads. Subsequent calls are warm.

For SRE-volume use (ingestion of a few hundred chunks per day,
on-the-fly embedding for query), this is plenty. If a consumer ever
wants to embed millions of documents quickly, a GPU-backed embedding
deployment (e.g. vLLM with `--task embed`) on a spare time-slice
share is worth standing up — but only when actually needed.

## What happens if it's down

- `rag-search` returns 502 `embedder unreachable` on both upsert and
  search paths.
- The Ollama-shim's `/api/embed` returns 502 to external callers.
- LLM chat endpoints are unaffected — the embeddings model is independent.

A restart is graceful: the next request after the pod comes back
re-loads the model (~10–15 seconds) and resumes. The PVC keeps the
model weights so this doesn't re-pull from Ollama's registry.

## Adding a second embedding model

`bge-m3` is the only embedding model the cluster offers today. If you
need a different one (e.g. higher dimension, language-specialized):

1. **Stand up a separate deployment** with a different model name. Don't
   mix two models behind the same service — clients can't tell them
   apart.
2. Add a new ClusterIP service for it.
3. Either teach `rag-search` to switch models via an env var, or
   stand up a second `rag-search` instance pointed at the new
   embedding service with a different `VECTOR_DIM`.

**Don't mix embedding models within a single collection.** All vectors
in a collection must come from the same model, or similarity math is
meaningless.

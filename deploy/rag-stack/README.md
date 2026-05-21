# RAG support stack — pgvector + Ollama bge-m3

CPU-only retrieval pipeline for SRE knowledgebase grounding. The agent layer
will embed runbooks/wiki/Confluence content into pgvector, then retrieve
matching chunks at query time to give the reasoning models facts to cite.

## Why these choices on this cluster

The first attempt at Qdrant + TEI failed on the GH200 nodes:

- Qdrant's bundled jemalloc aborts on the GH200's 64K kernel pages
  (the nodes run kernel `6.8.0-1032-nvidia-64k`). MALLOC_CONF workarounds
  did not bypass the issue. Tested with `qdrant/qdrant:v1.13.0` and `:latest`.
- Hugging Face TEI's CPU image (`ghcr.io/huggingface/text-embeddings-inference:cpu-1.5`
  and `:cpu-latest`) does not include a `linux/arm64` variant in its manifest.

Replaced with proven-on-this-cluster alternatives:

- **pgvector** (PostgreSQL 17 + pgvector extension) — multi-arch, ARM64-64K
  proven, mature HNSW indexes. Good to tens of millions of vectors.
- **Ollama** serving `bge-m3` — Ollama was already running fine on this
  cluster for 17 days before this migration, so platform compatibility
  is known-good.

## Components

| Component | Image | Purpose | Resources |
|---|---|---|---|
| pgvector | `pgvector/pgvector:pg17` | Vector DB + relational metadata | 2-4 CPU, 4-16 Gi RAM, 50 Gi PVC |
| `embeddings-ollama` | `ollama/ollama:latest` (CPU only) | `bge-m3` embeddings via Ollama `/api/embed` | 4-8 CPU, 8-16 Gi RAM, 20 Gi PVC |

A reranker is **not currently deployed** — see "Reranking" below.

## Internal endpoints

- pgvector: `postgresql://rag:CHANGE_ME_BEFORE_APPLY@domains-pgvector.lab-domains-sre.svc.cluster.local:5432/rag`
- Embeddings: `http://domains-embeddings.lab-domains-sre.svc.cluster.local:11434/api/embed`

ClusterIP only. The agent orchestrator inside the cluster talks to them.

## Reranking

Not deployed in this round. Two acceptable paths going forward:

1. Skip the dedicated reranker and use **reciprocal rank fusion** in the
   agent: combine BM25 from PostgreSQL `tsvector` with pgvector cosine
   similarity. This gives most of the quality benefit of a cross-encoder
   reranker without the extra service.
2. Later, add a small Python sentence-transformers pod serving
   `BAAI/bge-reranker-v2-m3` once we have the agent layer ready.

## Apply

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' apply -k 'C:\Users\fash\Desktop\AI Nodes\deploy\rag-stack'
```

## Scale up

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre scale deploy/pgvector --replicas=1
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre scale deploy/embeddings-ollama --replicas=1
```

## Bootstrap

After pgvector is Ready, create the extension and a chunks table:

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre exec deploy/pgvector -- psql -U rag -d rag -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

## Sanity check

Test the embeddings endpoint with a port-forward:

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre port-forward svc/domains-embeddings 11434:11434
# then in another shell:
# curl http://localhost:11434/api/embed -d '{"model":"bge-m3","input":"alert acknowledged"}'
```

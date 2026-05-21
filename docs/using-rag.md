# Using the RAG service

`rag-search` at `http://aicompute01.cnco1.tucows.cloud:31445` is a
generic embeddings + pgvector wrapper. Any team can put their own
knowledge base into a separate **collection** and query it
independently. It's the recommended way to give an LLM-powered app
grounded, citation-friendly retrieval without writing SQL or shipping a
vector-DB client.

## Concepts

**Collection.** A named bucket of documents. Each collection is a
separate PostgreSQL table (`rag_<name>`) so writes from one tenant
never touch another tenant's rows. Names must match
`^[a-z][a-z0-9_]{0,49}$`.

**Item.** A row in a collection. Has:

- `id` (BIGINT, you choose it — primary key, upsert semantics)
- `text` (the content you want to be searchable)
- `metadata` (free-form JSONB you can filter on at query time)
- `embedding` (1024-dim `bge-m3` vector, populated by the service —
  you never see this directly)

**Chunking.** rag-search does **not** chunk for you. Long documents
should be split into ~500-token chunks before calling `upsert`. The
service treats each `id` as one indivisible unit at query time.

A common pattern: use deterministic IDs like `(parent_doc_id * 1000) +
chunk_index`. Then re-running ingestion just overwrites the same rows.

## API reference

All endpoints are POST except where noted. Bodies are JSON. The base URL
is `http://aicompute01.cnco1.tucows.cloud:31445`.

### Liveness

```
GET /
```

Returns `{"status":"ok"}` if rag-search can reach pgvector, otherwise
`{"status":"degraded","pg_error":"..."}` with status 503.

### Create a collection

```
POST /v1/collections/{name}/init
```

Idempotent. Creates the table, the HNSW index on `embedding`, and the
GIN index on `metadata`. Safe to call at the start of every ingest job.

Response: `{"collection":"sre_runbooks","status":"ready"}`.

### Upsert one item

```
POST /v1/collections/{name}/upsert
{
  "id": 1234,
  "text": "Full text of the document or chunk that we want to be searchable.",
  "metadata": {
    "service": "checkout-api",
    "severity": "high",
    "source_url": "https://wiki.example.com/page/1234"
  }
}
```

The service embeds `text` via `bge-m3` and stores everything. If the id
already exists, the row is updated atomically (`ON CONFLICT (id) DO
UPDATE`).

Response: `{"id":1234,"status":"ok"}`.

### Upsert in bulk

```
POST /v1/collections/{name}/upsert_batch
{
  "items": [
    {"id": 1, "text": "...", "metadata": {...}},
    {"id": 2, "text": "...", "metadata": {...}}
  ]
}
```

Sequential upsert (not a database batch). Each item is embedded
individually. Use this to keep your HTTP overhead low when ingesting
many items at once; cap batches at ~25–50 items to keep request bodies
reasonable.

### Search

```
POST /v1/collections/{name}/search
{
  "query": "alert text or question",
  "top_k": 10,
  "filter": {"service": "checkout-api"}
}
```

- `top_k` defaults to 10, capped at 100.
- `filter` is optional. When present, it's applied as a JSONB
  containment match: `metadata @> filter`. Use it for exact-match
  filtering on metadata fields (`service`, `severity`, etc.).
- `query` is embedded and matched by **cosine similarity** against the
  collection's `embedding` column, using the HNSW index.

Response:

```json
{
  "results": [
    {
      "id": 42,
      "text": "...",
      "metadata": {"service": "checkout-api", "severity": "high"},
      "score": 0.8731
    },
    ...
  ]
}
```

`score` is `1 - cosine_distance`, so higher is more similar. Typical
thresholds: above 0.5 is "clearly relevant", above 0.7 is "almost
certainly the right answer", below 0.3 is "noise."

### Delete

```
POST /v1/collections/{name}/delete
{"ids": [42, 43, 44]}
```

Response: `{"deleted": 3}`.

### Count

```
GET /v1/collections/{name}/count
```

Response: `{"count": 241}`. Useful for sanity-checking ingestion.

## Recipes

### Ingest a small knowledge base

```python
import requests, json

BASE = "http://aicompute01.cnco1.tucows.cloud:31445"
COLL = "my_team_kb"

# 1. ensure the collection exists
requests.post(f"{BASE}/v1/collections/{COLL}/init", json={})

# 2. upsert in batches
docs = [
    {"id": 1, "text": "When the queue backs up, scale workers.",
     "metadata": {"service": "queue", "tags": ["scaling"]}},
    {"id": 2, "text": "Database CPU > 80% means review slow queries.",
     "metadata": {"service": "db",    "tags": ["performance"]}},
    # ... many more
]
for i in range(0, len(docs), 25):
    batch = docs[i:i+25]
    r = requests.post(f"{BASE}/v1/collections/{COLL}/upsert_batch",
                      json={"items": batch}, timeout=300)
    r.raise_for_status()

print("count:", requests.get(f"{BASE}/v1/collections/{COLL}/count").json())
```

### Search with a filter

```python
r = requests.post(f"{BASE}/v1/collections/{COLL}/search", json={
    "query": "what should I do when queues back up",
    "top_k": 5,
    "filter": {"service": "queue"},   # only match queue-related docs
})
for hit in r.json()["results"]:
    print(f"{hit['score']:.3f}  {hit['text'][:80]}")
```

### Citation-grounded LLM prompt

The whole point of retrieval is to ground LLM answers in facts. A
common pattern:

```python
from openai import OpenAI
llm = OpenAI(base_url="http://aicompute01.cnco1.tucows.cloud:31440/v1",
             api_key="x")

def answer_with_citations(question, collection="sre_runbooks"):
    hits = requests.post(
        f"{BASE}/v1/collections/{collection}/search",
        json={"query": question, "top_k": 5},
    ).json()["results"]

    context = "\n\n".join(
        f"[{i+1}] (id={h['id']}, score={h['score']:.2f})\n{h['text']}"
        for i, h in enumerate(hits)
    )

    msg = (
        "Answer using ONLY the context below. If the context does not "
        "contain the answer, say 'I don't know — not in the knowledge "
        "base.' Cite each fact with its [n] number.\n\n"
        f"## Context\n{context}\n\n## Question\n{question}"
    )

    resp = llm.chat.completions.create(
        model="qwen3-32b-thinking",
        messages=[{"role": "user", "content": msg}],
        extra_body={"chat_template_kwargs": {"enable_thinking": False}},
    )
    return resp.choices[0].message.content or resp.choices[0].message.reasoning_content

print(answer_with_citations("how do I restart zabbix agents?"))
```

### Hybrid keyword + dense search via RRF

Pure dense search misses exact-string matches sometimes (e.g. a UUID or
a specific error code). A common improvement: run a keyword scorer
against your source database (BM25 or whatever you already have), run
dense search against rag-search, then combine via reciprocal rank
fusion:

```python
K = 60  # RRF constant; 60 is the conventional default

def rrf_merge(keyword_hits, dense_hits, top_n=10):
    """keyword_hits and dense_hits are lists of ids in rank order."""
    scores = {}
    for rank, hid in enumerate(keyword_hits, start=1):
        scores[hid] = scores.get(hid, 0) + 1.0 / (K + rank)
    for rank, hid in enumerate(dense_hits, start=1):
        scores[hid] = scores.get(hid, 0) + 1.0 / (K + rank)
    return sorted(scores.items(), key=lambda x: -x[1])[:top_n]
```

This is exactly how UIP's `runbook-api` `/api/runbook/match` ranks
results — see [components/rag-search.md](components/rag-search.md) for
the full pattern.

## Existing collections

Maintained centrally; talk to the owner before writing to a collection
you don't own.

| Collection | Owner | Schema |
|---|---|---|
| `sre_runbooks` | UIP runbook-api | id=runbook SQLite row id; metadata: `{hostname, service, severity}` |
| `occ_wiki` | wiki-ingester CronJob | id=page_id*1000+chunk_index; metadata: `{page_id, page_title, page_url, chunk_index, chunk_heading, source, space_key, updated_at}` |

If you want your own collection, pick a name that says who owns it
(`yourteam_runbooks`, `yourteam_docs`) and just call `/init`. No
coordination needed — the table is created on demand.

## Limits and behavior

- **One row per id.** Upserts overwrite. There is no versioning. If you
  need history, store it as part of `text` or `metadata`.
- **No bulk delete-by-filter.** You must delete by id list. If you need
  to clear a collection, `kubectl exec deploy/pgvector -- psql -U rag -d
  rag -c "TRUNCATE rag_yourcollection"`.
- **No automatic cleanup of stale rows.** Deletions in the source
  system do not propagate. The wiki-ingester re-runs daily and re-upserts
  current pages, but deleted Confluence pages linger in `occ_wiki`.
  Address it when it bites; not before.
- **Embedding model is hard-coded.** All collections use `bge-m3`
  (1024-dim). Mixing embedding models within a collection breaks
  similarity math. If you need a different model, request a separate
  cluster-wide service — don't try to swap it per collection.
- **No streaming.** Responses are bounded JSON — search results are
  returned all at once.

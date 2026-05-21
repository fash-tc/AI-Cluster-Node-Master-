# rag-search

The generic embeddings + pgvector retrieval service. The recommended
way to give any LLM-powered app grounded, citation-friendly retrieval
without writing SQL or shipping a vector-DB client.

## What it is

- **Image:** `python:3.12-slim` + `psycopg[binary]==3.2.3` (installed
  into a PVC at first start, cached for subsequent restarts)
- **Code:** [`deploy/rag-search/configmap.yaml`](../../deploy/rag-search/configmap.yaml)
- **Bundle:** [`deploy/rag-search/`](../../deploy/rag-search/)
- **Resources:** 200m-2 CPU, 128-512 MiB RAM

## Where it runs

Single-replica deployment, lands on whichever node has spare CPU
capacity. Stateless — can scale horizontally behind the same NodePort
if traffic ever warrants it. The actual data lives in pgvector, not
inside rag-search.

## Architecture

```
client → POST /v1/collections/{name}/{op} → rag-search
                                              │
                                              ├─ embed: POST /api/embed → embeddings-ollama (bge-m3)
                                              │
                                              └─ store / search: SQL → pgvector
```

## API summary

See [docs/using-rag.md](../using-rag.md) for the full API reference with
examples. Quick recap:

| Path | Method | Purpose |
|---|---|---|
| `/` | GET | Liveness (checks pgvector connectivity) |
| `/v1/collections/{name}/init` | POST | Idempotent table + HNSW + GIN index create |
| `/v1/collections/{name}/upsert` | POST | Embed text, store one row |
| `/v1/collections/{name}/upsert_batch` | POST | Sequential upsert of `items: [...]` |
| `/v1/collections/{name}/search` | POST | Embed query, return top-k by cosine |
| `/v1/collections/{name}/delete` | POST | Remove rows by `ids: [...]` |
| `/v1/collections/{name}/count` | GET | Row count |

## Endpoints

- External: `http://aicompute01.cnco1.tucows.cloud:31445`
- In-cluster: `http://domains-rag-search.lab-domains-sre.svc.cluster.local:8081`

## Per-collection schema

Each `POST /init` creates a table:

```sql
CREATE TABLE rag_<name> (
  id BIGINT PRIMARY KEY,
  text TEXT NOT NULL,
  metadata JSONB DEFAULT '{}'::jsonb,
  embedding vector(1024),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX rag_<name>_embedding_idx ON rag_<name> USING hnsw (embedding vector_cosine_ops);
CREATE INDEX rag_<name>_metadata_idx ON rag_<name> USING gin (metadata);
```

The HNSW index speeds up similarity search; the GIN index speeds up
JSONB containment filters. Both are critical for query performance at
scale (>~10K rows).

## Collection naming

Must match `^[a-z][a-z0-9_]{0,49}$`. Validated at every request to
prevent SQL injection through the path parameter. (The service does
this validation in Python before any SQL is built — see the
`_COLLECTION_RE` regex in the source.)

By convention, prefix names with the owning team or app
(`yourteam_kb`, `yourapp_docs`). Collisions are unlikely with a
descriptive prefix and there is no enforcement.

## Discovering what's live

The service does not have a global "list collections" endpoint, so
discovery is via PostgreSQL directly:

```bash
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  psql -U rag -d rag -c "\dt rag_*"
```

That lists every collection table. Use
`GET /v1/collections/{name}/count` to see row counts per collection.

## Embedding dimension

Hard-coded to **1024** (`VECTOR_DIM=1024` env). This matches `bge-m3`.
If you swap embedding models, both `VECTOR_DIM` and the table schema
must change in lockstep — and every existing row must be re-embedded.
Don't do this lightly.

## Credentials

The service uses a Secret named `rag-search-pg-dsn` containing the
PostgreSQL connection string. Today it's the shared `rag` role
(read/write on all `rag_*` tables in the `rag` database).

For per-tenant isolation, see
[operations.md](../operations.md#hardening-pgvector-for-multi-tenant-use).

## Error handling

The service returns standard HTTP status codes:

- `200` — success
- `400` — client error (invalid collection name, bad JSON, etc.)
- `404` — unknown path or unknown operation
- `500` — internal PostgreSQL error (table doesn't exist, deadlock, etc.)
- `502` — embeddings backend (`embeddings-ollama`) is unreachable
- `503` — pgvector is unreachable (returned from `/` liveness check)

Each error response is `{"error": "...message..."}`.

## Adding the service to a new tenant's workflow

Three steps for a new team:

1. **Pick a collection name** (e.g. `team_x_kb`). Add it to your
   service's config.
2. **Call `/v1/collections/team_x_kb/init`** at app startup. It's
   idempotent — safe to call on every boot.
3. **Read/write** via the API. Don't share the collection name with
   anyone outside the team.

That's all there is to it. No K8s changes needed — the service
auto-creates the underlying table on first init.

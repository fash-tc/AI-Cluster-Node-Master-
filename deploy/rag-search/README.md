# rag-search

Generic embeddings + pgvector retrieval service. Wraps bge-m3 (Ollama) and
pgvector behind a small REST API so multiple teams can each manage their own
collection without touching SQL or vector libraries.

## Endpoints

| Method | Path | Body | Purpose |
|---|---|---|---|
| GET | `/` | — | Liveness; verifies pgvector reachable |
| POST | `/v1/collections/{name}/init` | — | Idempotent create-table + HNSW + GIN(metadata) index |
| POST | `/v1/collections/{name}/upsert` | `{id, text, metadata?}` | Embed `text` via bge-m3, upsert row |
| POST | `/v1/collections/{name}/upsert_batch` | `{items: [{id, text, metadata?}, …]}` | Sequential upsert of many |
| POST | `/v1/collections/{name}/search` | `{query, top_k?, filter?}` | Embed `query`, return top-k by cosine similarity; optional `filter` is a JSONB `@>` match against `metadata` |
| POST | `/v1/collections/{name}/delete` | `{ids: [int, …]}` | Remove rows by id |
| GET | `/v1/collections/{name}/count` | — | Row count |

Collection names must match `^[a-z][a-z0-9_]{0,49}$`. Each collection becomes a
table `rag_<name>` in the shared `rag` PostgreSQL database.

## Endpoint URLs

- External: `http://aicompute01.cnco1.tucows.cloud:31445`
- In-cluster: `http://domains-rag-search.lab-domains-sre.svc.cluster.local:8081`

## Multi-tenant pattern

Each consumer owns a collection name. Examples of how teams might pick
names:

| Collection (example) | Purpose |
|---|---|
| `yourteam_kb` | Your team's general knowledge base |
| `yourapp_docs` | Documentation for a specific app |
| `project_runbooks` | Project-scoped operational notes |

Collections share the embedding model and pgvector instance but have isolated
tables. Searches against one collection cannot surface rows from another.

For stronger isolation, add a per-team PostgreSQL role + schema (the secret
in this bundle uses the shared `rag` user — fine while consumers are trusted,
but worth hardening once external teams come on board).

## Vector dimensions

`bge-m3` produces 1024-dim embeddings, so the `vector(1024)` column type is
hardcoded. If you swap to a different embedding model later, both
`VECTOR_DIM` and the schema have to change in lockstep.

## Apply paused

```bash
kubectl apply -k deploy/rag-search
```

## Scale up

```bash
kubectl -n lab-domains-sre scale deploy/rag-search --replicas=1
```

First start installs `psycopg[binary]==3.2.3` into a 1 GiB PVC. Subsequent
restarts skip the install and come up in seconds.

## Sanity test

```bash
# Init a collection
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/test_kb/init

# Upsert one entry
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/test_kb/upsert \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "text": "The capital of France is Paris.", "metadata": {"category": "geography"}}'

# Search
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/test_kb/search \
  -H "Content-Type: application/json" \
  -d '{"query": "what is the French capital", "top_k": 5}'
```

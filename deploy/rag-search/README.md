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

- External (UIP host, etc.): `http://aicompute01.cnco1.tucows.cloud:31445`
- In-cluster: `http://domains-rag-search.lab-domains-sre.svc.cluster.local:8081`

## Multi-tenant pattern

Each consumer owns a collection name:

| Collection | Owner | Purpose |
|---|---|---|
| `sre_runbooks` | UIP runbook-api | Alert remediation notes |
| `team_x_docs` | (future) | Other team's knowledgebase |

Collections share the embedding model and pgvector instance but have isolated
tables. Searches against `sre_runbooks` cannot accidentally surface rows from
`team_x_docs`.

For stronger isolation, add a per-team PostgreSQL role + schema (the secret
in this bundle uses the shared `rag` user — fine while consumers are trusted,
but worth hardening once external teams come on board).

## Vector dimensions

`bge-m3` produces 1024-dim embeddings, so the `vector(1024)` column type is
hardcoded. If you swap to a different embedding model later, both
`VECTOR_DIM` and the schema have to change in lockstep.

## Apply paused

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' apply -k 'C:\Users\fash\Desktop\AI Nodes\deploy\rag-search'
```

## Scale up

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre scale deploy/rag-search --replicas=1
```

First start installs `psycopg[binary]==3.2.3` into a 1 GiB PVC. Subsequent
restarts skip the install and come up in seconds.

## Sanity test

```bash
# Init a collection
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/sre_runbooks/init

# Upsert one entry
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/sre_runbooks/upsert \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "text": "When checkout-api 5xx spikes, check redis hit rate first then postgres slow log.", "metadata": {"service": "checkout-api", "severity": "high"}}'

# Search
curl -X POST http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/sre_runbooks/search \
  -H "Content-Type: application/json" \
  -d '{"query": "checkout latency", "top_k": 5}'
```

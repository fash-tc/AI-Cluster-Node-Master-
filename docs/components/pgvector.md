# pgvector

The vector store. PostgreSQL 17 with the `vector` extension. Holds all
RAG-collection data for every consumer.

## What it is

- **Image:** `pgvector/pgvector:pg17`
- **Database:** `rag`, owned by user `rag`
- **Extension:** `vector` version 0.8.x (installed at first start via
  `CREATE EXTENSION IF NOT EXISTS vector` from `rag-search`)
- **Bundle:** [`deploy/rag-stack/`](../../deploy/rag-stack/) (same bundle
  as `embeddings-ollama`)
- **Resources:** 2-4 CPU, 4-16 GiB RAM, 50 GiB local-path PVC

## Where it runs

Single-replica deployment. Lands on whichever compute node has the
`pgvector-data` PVC bound — once it binds (first scheduling) the PVC is
node-pinned for the lifetime of the PVC. The pod stays there.

`pgvector` is ClusterIP-only by design. Reachable from inside the
cluster at:

```
domains-pgvector.lab-domains-sre.svc.cluster.local:5432
```

No NodePort. No external access. The only consumer is `rag-search`
(plus the operator via `kubectl exec` for admin tasks).

## Why pgvector, not Qdrant/Weaviate/etc.

The original design picked Qdrant. Qdrant crashed at startup on this
cluster because its bundled jemalloc isn't compiled for 64K kernel
pages. Pin or `:latest` — same crash both ways.

PostgreSQL on aarch64 handles 64K pages cleanly without any special
configuration. The `pgvector` extension adds a `vector` column type and
HNSW + IVFFlat index types. For tens of millions of vectors, it
performs well enough that we don't miss a dedicated vector DB.

The trade-offs vs. a dedicated vector DB:

| Aspect | pgvector | Dedicated (Qdrant/Weaviate) |
|---|---|---|
| ARM64 + 64K pages | Works | Often broken |
| HNSW recall/speed | Good | Marginally better at huge scale (>50M) |
| Filtering by metadata | JSONB `@>` + GIN index — very flexible | Per-vector-DB syntax |
| Operations (backup, monitoring) | Standard Postgres tooling | Vendor-specific |
| Hybrid (dense + sparse) | Manual via Postgres tsvector + pgvector | First-class in some DBs |

For our workload — SRE runbooks, wiki ingestion, occasional teams
adding their own KBs — pgvector is the right call.

## Schema

Every rag-search collection materializes as a table:

```sql
CREATE TABLE rag_<collection_name> (
  id          BIGINT PRIMARY KEY,
  text        TEXT NOT NULL,
  metadata    JSONB DEFAULT '{}'::jsonb,
  embedding   vector(1024),
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX rag_<collection_name>_embedding_idx
  ON rag_<collection_name> USING hnsw (embedding vector_cosine_ops);
CREATE INDEX rag_<collection_name>_metadata_idx
  ON rag_<collection_name> USING gin (metadata);
```

Tables are created on demand by rag-search's `/v1/collections/{name}/init`
endpoint.

## Common admin operations

Connect to the database directly:

```bash
kubectl -n lab-domains-sre exec -it deploy/pgvector -- psql -U rag -d rag
```

Once in psql:

```sql
-- List collection tables
\dt rag_*

-- Row counts per collection
SELECT relname AS table, n_live_tup AS rows
  FROM pg_stat_user_tables
  WHERE relname LIKE 'rag_%'
  ORDER BY n_live_tup DESC;

-- Peek at a collection's contents
SELECT id, LEFT(text, 80) AS preview, metadata->>'category' AS category
  FROM rag_<your_collection>
  ORDER BY updated_at DESC
  LIMIT 5;

-- Truncate a collection (no undo — be sure)
TRUNCATE rag_<your_collection>;

-- Drop a collection entirely
DROP TABLE rag_<your_collection>;

-- Check HNSW index health
\d+ rag_<your_collection>
```

## Backup

There is no automated backup. Run periodic `pg_dump` if you can't
afford to re-ingest from scratch:

```bash
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  pg_dump -U rag -d rag --clean --if-exists \
  > pgvector-$(date +%F).sql
```

Restore from inside the pod:

```bash
kubectl -n lab-domains-sre cp pgvector-2026-05-20.sql lab-domains-sre/pgvector-XXX:/tmp/restore.sql
kubectl -n lab-domains-sre exec deploy/pgvector -- psql -U rag -d rag -f /tmp/restore.sql
```

## Capacity

The PVC is **50 GiB**. Approximate sizing:

- A single `vector(1024)` row with FP4 storage costs ~512 bytes
- Plus ~1–4 KB of `text` + `metadata` per row depending on chunk size
- HNSW indexes cost ~20–40% on top of base data
- → roughly 5 KB per row at SRE-typical chunk sizes

50 GiB / 5 KB ≈ **10 million vectors**. We're at ~290 today across all
collections. Plenty of runway. Bump the PVC size when projected growth
makes 50 GiB feel uncomfortable.

## Credentials

Stored in the K8s Secret `pgvector-credentials`. The default placeholder
in this repo is `CHANGE_ME_BEFORE_APPLY` — substitute a real value
before applying in any environment you care about. The same value goes
into `rag-search-pg-dsn` so the consumer can connect.

To rotate:

```bash
# Update Postgres
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  psql -U rag -d rag -c "ALTER USER rag WITH PASSWORD 'newpass';"

# Update K8s Secrets
kubectl -n lab-domains-sre delete secret pgvector-credentials rag-search-pg-dsn
kubectl -n lab-domains-sre create secret generic pgvector-credentials \
  --from-literal=POSTGRES_USER=rag \
  --from-literal=POSTGRES_PASSWORD=newpass \
  --from-literal=POSTGRES_DB=rag
kubectl -n lab-domains-sre create secret generic rag-search-pg-dsn \
  --from-literal=dsn='postgresql://rag:newpass@domains-pgvector.lab-domains-sre.svc.cluster.local:5432/rag'

# Restart consumers
kubectl -n lab-domains-sre rollout restart deploy/rag-search
```

(pgvector itself doesn't need a restart — Postgres `ALTER USER`
takes effect on the next connection.)

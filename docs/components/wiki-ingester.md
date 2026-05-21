# wiki-ingester

The Confluence → RAG sync job. A K8s CronJob that pulls the OCC
Confluence space and upserts every page into the `occ_wiki` collection.

## What it is

- **Image:** `python:3.12-slim` + `html2text==2024.2.26` (pip-installed
  into a PVC at first run, cached afterwards)
- **Code:** [`deploy/wiki-ingester/configmap.yaml`](../../deploy/wiki-ingester/configmap.yaml)
- **Bundle:** [`deploy/wiki-ingester/`](../../deploy/wiki-ingester/)
- **Schedule:** Daily at **03:30 UTC**
- **Resources:** 100m-1 CPU, 128-512 MiB RAM, 256 MiB PVC

## Where it runs

Runs as a Kubernetes Job triggered by a CronJob. Each run is a fresh
pod, scheduled on whichever node has capacity. The PVC
(`wiki-ingester-pydeps`) caches the pip-installed `html2text` package
so subsequent runs start quickly.

## What it does

1. Authenticate to Atlassian via Basic Auth (email:token from K8s Secret)
2. `GET /wiki/api/v2/spaces?keys=OCC&limit=5` — resolve the space ID
3. `GET /wiki/api/v2/spaces/{id}/pages?body-format=storage&limit=50`
   (paginated) — list all current pages with storage XHTML body
4. For each page:
   - Convert Confluence storage XHTML to plain text via `html2text`
     (with custom pre-processing to strip `<ac:*>` macros and
     `<ri:*>` resource references)
   - Chunk by heading + paragraph (~2200 chars per chunk, 300-char
     minimum, merge tiny adjacent chunks)
   - `POST /v1/collections/occ_wiki/upsert_batch` to rag-search, 25
     chunks per batch
5. Print a per-page summary line; exit 0 on success, 1 on any failure

## Idempotency

Chunk IDs are deterministic: `page_id * 1000 + chunk_index`. Re-running
the ingester just overwrites existing rows with refreshed content via
`ON CONFLICT (id) DO UPDATE`. There is no risk of duplicate rows or
ID collisions.

Pages get the same ID every run, so an edit in Confluence becomes
visible in the next ingester run (within 24 hours by default — sooner
if triggered manually).

## What it does NOT do

- **Delete pruning.** Pages deleted from Confluence linger in the
  `occ_wiki` collection. The ingester doesn't track current vs. previous
  page sets. Fix this if it ever bites by adding a diff step before
  the upsert loop.
- **Attachment ingestion.** Only page body text is indexed. Images,
  files, embedded charts are ignored.
- **Comment ingestion.** Confluence inline and footer comments are not
  indexed.
- **Cross-space sync.** Only the OCC space is configured. To add
  another space, add a second CronJob (or parameterize this one) — don't
  mix spaces in the same collection.

## Endpoints

The ingester is internal-only — it has no service, no NodePort. It
talks to:

- **Atlassian REST API** at `https://wiki-tucows.atlassian.net/wiki/api/v2/...`
  (outbound HTTPS from the cluster)
- **rag-search** at `http://domains-rag-search.lab-domains-sre.svc.cluster.local:8081`
  (in-cluster HTTP)

## Triggering manually

Reasons to manually trigger:

- Right after editing a Confluence page that needs to be searchable
  immediately
- Testing the ingester after a config change
- Initial bulk fill after deploying the bundle

Command:

```bash
JOB="wiki-ingester-manual-$(date +%Y%m%d%H%M)"
kubectl -n lab-domains-sre create job --from=cronjob/wiki-ingester $JOB
kubectl -n lab-domains-sre logs -f -l job-name=$JOB
```

Expect ~3 minutes for a full re-ingest of ~25 pages.

## Credentials

Stored in K8s Secret `wiki-ingester-creds` with keys `ATLASSIAN_EMAIL`
and `ATLASSIAN_API_TOKEN`. The secret is intentionally **excluded from
the Kustomize bundle** (`secret.yaml` is in the deploy dir as a
template but not listed under `resources:`). Create it out-of-band:

```bash
kubectl -n lab-domains-sre create secret generic wiki-ingester-creds \
  --from-literal=ATLASSIAN_EMAIL='your.email@tucows.com' \
  --from-literal=ATLASSIAN_API_TOKEN='<paste-token-here>'
```

Token requirements:

- Created at <https://id.atlassian.com/manage-profile/security/api-tokens>
- Must have **Confluence read** scope (a Jira-only token returns 403 —
  see [gotchas](../gotchas.md#atlassian-api-token-must-have-confluence-scope))
- Email must be the user's **Confluence** email, not their Jira email
  if they differ (see [gotchas](../gotchas.md#atlassian-email-differs-by-product))

To rotate the token:

```bash
kubectl -n lab-domains-sre delete secret wiki-ingester-creds
kubectl -n lab-domains-sre create secret generic wiki-ingester-creds \
  --from-literal=ATLASSIAN_EMAIL='...' \
  --from-literal=ATLASSIAN_API_TOKEN='...'
# The CronJob picks up the new secret on its next run; no restart needed.
```

## Configuration

| Env var | Default | What it does |
|---|---|---|
| `CONFLUENCE_SITE` | `https://wiki-tucows.atlassian.net` | Atlassian instance URL |
| `SPACE_KEY` | `OCC` | Which space to ingest |
| `RAG_BASE` | `http://domains-rag-search.lab-domains-sre.svc.cluster.local:8081` | Where to write |
| `COLLECTION` | `occ_wiki` | Collection name in rag-search |
| `CHUNK_TARGET_CHARS` | `2200` | Target chunk size (~500 tokens for bge-m3) |
| `CHUNK_MIN_CHARS` | `300` | Tiny chunks are merged back into neighbors |

Override in `cronjob.yaml` and re-apply if you need different values.

## "Failed" status that isn't really a failure

The job exits 1 if **any** page fails to upsert. A single transient
502 from rag-search during a long run causes the whole job to be
marked Failed. Always verify the actual data before treating Failed as
an outage:

```bash
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  psql -U rag -d rag -c "SELECT COUNT(*) FROM rag_occ_wiki;"
```

If the count looks right, re-trigger manually and the missed pages
land on the retry.

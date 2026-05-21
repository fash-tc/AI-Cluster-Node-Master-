# Confluence → RAG ingester

K8s CronJob that mirrors a Confluence space into a rag-search collection.

Configurable per deployment via env vars in `cronjob.yaml`:

- `CONFLUENCE_SITE` — Atlassian instance URL
- `SPACE_KEY` — which Confluence space to ingest (the short key)
- `COLLECTION` — rag-search collection name to write into

To run multiple ingesters (different spaces → different collections),
copy this bundle and rename the resources. See
[docs/components/wiki-ingester.md](../../docs/components/wiki-ingester.md#deploying-a-second-instance).

- **Schedule:** daily at 03:30 UTC (edit `schedule:` in `cronjob.yaml`
  if you want a different cadence)
- **Scope:** all current pages in the configured space, converted from
  Confluence storage XHTML to plain text and chunked by heading +
  paragraph
- **Idempotent:** chunk IDs are `page_id * 1000 + chunk_index` so
  re-runs overwrite existing rows with refreshed content; deleted pages
  are NOT automatically pruned (see "Caveats")

## Apply

The Secret holds your Atlassian email + API token. **Don't commit the
populated file.** The bundle's `kustomization.yaml` intentionally
excludes `secret.yaml` so the placeholder never gets pushed to Git.
Create the secret out-of-band:

```bash
kubectl -n lab-domains-sre create secret generic wiki-ingester-creds \
  --from-literal=ATLASSIAN_EMAIL='your.email@example.com' \
  --from-literal=ATLASSIAN_API_TOKEN='<your-atlassian-api-token>'
```

Then apply the rest:

```bash
kubectl apply -k deploy/wiki-ingester
```

Get an Atlassian API token at
[id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens).
The token needs Confluence read access for the target space (a
Jira-only token returns 403 — see
[docs/gotchas.md](../../docs/gotchas.md#atlassian-api-token-must-have-confluence-scope)).

## Trigger a one-off bulk run

The CronJob fires at 03:30 UTC. To run the ingester immediately
(initial fill after first deploy, manual re-sync after editing pages,
troubleshooting):

```bash
JOB="wiki-ingester-bulk-$(date +%Y%m%d%H%M)"
kubectl -n lab-domains-sre create job --from=cronjob/wiki-ingester "$JOB"
kubectl -n lab-domains-sre logs -f -l job-name="$JOB"
```

## Caveats

- **Deletions are not propagated.** If a page is deleted in Confluence,
  its chunks linger in the collection. Fix this when it becomes an
  issue by adding a sweep step at the end of the ingester that diffs
  current page IDs against existing rows in pgvector.
- **Token rotation.** When you rotate the Atlassian token, delete and
  recreate the Secret. The CronJob picks up the new value on its next
  run; no restart needed.
- **Storage format conversion is lossy.** Confluence-specific macros
  (info panels, inline Jira issues, etc.) are stripped before
  `html2text` runs. The semantic content survives; visual layout
  doesn't.

## Endpoint summary

| URL | What it does |
|---|---|
| `http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/<COLLECTION>/search` | RAG search over the configured wiki collection (via the multi-tenant `rag-search` service) |

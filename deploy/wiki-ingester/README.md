# Confluence OCC → RAG ingester

K8s CronJob that mirrors the OCC Confluence space into the `occ_wiki`
collection in rag-search.

- **Schedule:** daily at 03:30 UTC
- **Scope:** all current pages in the OCC space, converted from Confluence
  storage XHTML to plain text and chunked by heading + paragraph
- **Idempotent:** chunk IDs are `page_id * 1000 + chunk_index` so re-runs
  overwrite existing rows with refreshed content; deleted pages are NOT
  automatically pruned (see "Caveats")

## Apply

The Secret holds your Atlassian email + API token. **Don't commit the
populated file.** Either edit `secret.yaml` locally just before applying,
or skip it in `kustomization.yaml` and create the secret out-of-band:

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre create secret generic wiki-ingester-creds `
  --from-literal=ATLASSIAN_EMAIL='fash@tucowsinc.com' `
  --from-literal=ATLASSIAN_API_TOKEN='<your-atlassian-api-token>'
```

Then apply the rest:

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' apply -k 'C:\Users\fash\Desktop\AI Nodes\deploy\wiki-ingester'
```

Get an Atlassian API token at
[id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens).
The token only needs Confluence read access for the OCC space.

## Trigger a one-off bulk run

The CronJob fires at 03:30 UTC. To run the ingester immediately (initial
fill, manual re-sync, troubleshooting):

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre create job --from=cronjob/wiki-ingester wiki-ingester-bulk-$(Get-Date -Format yyyyMMddHHmm)
```

Watch progress:

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre logs -f -l job-name=wiki-ingester-bulk-…
```

## Caveats

- **Deletions are not propagated.** If a page is deleted in Confluence,
  its chunks linger in the `occ_wiki` collection. Fix this when it
  becomes an issue by adding a sweep step at the end of the ingester that
  diffs current page IDs against existing rows in pgvector.
- **Token rotation.** The API token in the Secret is yours. When you
  rotate it, run the `kubectl create secret … --dry-run=client -o yaml |
  kubectl apply -f -` dance to replace without restarting the CronJob.
- **Storage format conversion is lossy.** Confluence-specific macros
  (info panels, jira inline issues, etc.) are stripped before
  `html2text` runs. The semantic content survives; visual layout doesn't.

## Endpoint summary

| URL | Service |
|---|---|
| `http://aicompute01.cnco1.tucows.cloud:31445/v1/collections/occ_wiki/search` | RAG search over OCC wiki (multi-tenant `rag-search`) |

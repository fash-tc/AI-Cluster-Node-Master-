# Operations

Day-2 procedures: deploy, scale, observe, recover.

This doc assumes you have `kubectl` configured against the cluster and
`OS_TOKEN` set in your environment. See
[hardware.md](hardware.md#cluster-access) for the token flow.

## Apply a bundle

Every service in `deploy/` is a Kustomize bundle. Apply pattern:

```powershell
& 'C:\path\to\kubectl.exe' --kubeconfig 'C:\path\to\kubeconfig' apply -k 'C:\path\to\AI-Cluster-Node-Master-\deploy\<bundle>'
```

Every reasoning-model bundle ships with **`replicas: 0`** in its
`deployment.yaml`. Applying creates the resources but doesn't pull
weights or start inference. To bring it up:

```bash
kubectl --kubeconfig <kc> -n lab-domains-sre scale deploy/<name> --replicas=1
```

Scale to 0 to pause. Scale to 1 to resume. Same command both ways.

## Scaling and rolling restarts

### Scale a model up or down

```bash
# pause (no weights loaded, no GPU consumption)
kubectl -n lab-domains-sre scale deploy/qwen3-32b-thinking-aicompute02 --replicas=0

# resume
kubectl -n lab-domains-sre scale deploy/qwen3-32b-thinking-aicompute02 --replicas=1
```

First scale-up after a `replicas: 0` period takes 5–15 min for the 32B
(model pull + load) and 30–60 min for the 235B (220 GiB pull + load).

### Rolling restart (pick up new image or env)

```bash
kubectl -n lab-domains-sre rollout restart deploy/<name>
kubectl -n lab-domains-sre rollout status  deploy/<name> --timeout=30m
```

Note: most model bundles use `strategy: Recreate`, so a rolling restart
kills the old pod before starting the new one. There's a window of
service unavailability — the gateway's load balancer will fail over to
the other 32B replica if you're rolling one of them.

### Wait for a model to be available

```bash
kubectl -n lab-domains-sre wait \
  --for=condition=available --timeout=20m \
  deploy/qwen3-32b-thinking-aicompute02
```

## Watch what's happening

```bash
# Snapshot of everything
kubectl -n lab-domains-sre get all -o wide

# Continuously stream logs of one deployment
kubectl -n lab-domains-sre logs -f deploy/qwen3-32b-thinking-aicompute02

# Stream events (useful when pods are stuck Pending or CrashLooping)
kubectl -n lab-domains-sre get events --sort-by='.lastTimestamp' | tail -30

# All recent events for a specific pod
kubectl -n lab-domains-sre describe pod -l app=qwen3-32b-thinking
```

The vLLM servers log token throughput every 10 seconds when actively
serving — useful for confirming inference is happening.

## Deploy a brand-new bundle

The pattern is always:

1. Write the manifests under `deploy/<name>/`. Include at minimum
   `configmap.yaml` (or env vars in the deployment), `deployment.yaml`,
   `service.yaml`, and `kustomization.yaml`.
2. `kubectl kustomize deploy/<name>` to render and eyeball the output.
3. `kubectl apply -k deploy/<name> --dry-run=server` to validate against
   the cluster's admission.
4. Real apply: `kubectl apply -k deploy/<name>`.
5. Scale up only after step 4 succeeds.

Always start at `replicas: 0` for model workloads and scale up
explicitly. The cluster has historically had one node-NotReady
incident from a too-eager scale-up on a too-big model; the
paused-by-default discipline avoids repeating it.

## Manually trigger the wiki-ingester CronJob

The CronJob runs at 03:30 UTC daily. For an immediate run (after a doc
edit, or just to verify it works):

```powershell
$JOB="wiki-ingester-manual-$(Get-Date -Format yyyyMMddHHmm)"
kubectl -n lab-domains-sre create job --from=cronjob/wiki-ingester $JOB
kubectl -n lab-domains-sre logs -f -l job-name=$JOB
```

Bash equivalent:

```bash
JOB="wiki-ingester-manual-$(date +%Y%m%d%H%M)"
kubectl -n lab-domains-sre create job --from=cronjob/wiki-ingester $JOB
kubectl -n lab-domains-sre logs -f -l job-name=$JOB
```

The job is idempotent — chunk IDs are deterministic, so re-runs just
refresh content for pages whose markup changed.

## Secrets management

Two real secrets in the cluster today:

| Secret | What it holds | Created via |
|---|---|---|
| `pgvector-credentials` | PostgreSQL user/password/db for pgvector | `kubectl apply -k deploy/rag-stack` — change `pgvector-secret.yaml` before applying |
| `wiki-ingester-creds` | Atlassian email + API token | `kubectl create secret generic wiki-ingester-creds --from-literal=...` (out-of-band; the placeholder `secret.yaml` is intentionally excluded from kustomization) |

To rotate either:

```bash
kubectl -n lab-domains-sre delete secret <name>
kubectl -n lab-domains-sre create secret generic <name> --from-literal=KEY1=VALUE1 --from-literal=KEY2=VALUE2
kubectl -n lab-domains-sre rollout restart deploy/<consumer>
```

The consumer needs to be restarted because Kubernetes does not
automatically reload secret values into running pods.

## Hardening pgvector for multi-tenant use

Today, all rag-search collections share one PostgreSQL role (`rag`). When
a second team comes on board with its own collection, the right
hardening is:

```sql
-- in pgvector (kubectl exec deploy/pgvector -- psql -U rag -d rag)
CREATE ROLE team_x_rw WITH LOGIN PASSWORD '...';
CREATE SCHEMA team_x AUTHORIZATION team_x_rw;
-- Force rag-search to use schema-qualified table names like
-- "team_x.rag_kb" instead of "rag_kb". (Would require a rag-search
-- enhancement that today does not exist.)
```

The current rag-search code uses unqualified table names. Until that
gains schema support, tenancy is purely conventional (don't write to
collections you don't own). Add the enhancement when a second tenant
actually shows up.

## Disaster recovery

### Node goes NotReady mid-incident

1. **Stop adding load.** Don't scale up anything.
2. Confirm the node is actually NotReady: `kubectl get nodes`.
3. Check `kubectl describe node <name>` for `Conditions` and recent
   events.
4. If the node's underlying VM is dead (e.g. OpenStack issue), wait for
   it to recover or escalate to platform team. The K8s scheduler will
   not move PVCs off a dead node — local-path is node-pinned.
5. After recovery, scale up the affected deployments individually and
   watch logs.

### pgvector data corruption / accidental TRUNCATE

The data is on the `pgvector-data` PVC on whichever node it landed on.
There is no automatic backup. Run periodic `pg_dump` if the cost of
re-running ingestion across all consumers is unacceptable:

```bash
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  pg_dump -U rag -d rag --clean --if-exists > pgvector-dump-$(date +%F).sql
```

Restore with `psql -U rag -d rag -f pgvector-dump-YYYY-MM-DD.sql` from
within the pgvector pod.

### A whole model deployment is broken

Every bundle has a paused-by-default config (`replicas: 0`). Recovery:

```bash
# Roll back to paused
kubectl -n lab-domains-sre scale deploy/<broken> --replicas=0

# Fix the manifests, re-apply
kubectl apply -k deploy/<broken>

# Bring it back when ready
kubectl -n lab-domains-sre scale deploy/<broken> --replicas=1
```

Model weights stay in the PVC across pod replacements, so re-scaling
doesn't re-pull from Hugging Face.

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Pod stuck in `Pending` with no node assigned | nodeSelector specifies an unavailable node, or all GPU shares taken | `kubectl describe pod <name>`; look at scheduler events |
| vLLM exits with `User-specified max_model_len ... is greater than the derived max_model_len` | Configured context exceeds the model's native `max_position_embeddings` (no rope scaling) | Lower `MAX_MODEL_LEN` in the configmap to the model's native limit |
| Container `CrashLoopBackOff` with `jemalloc: Unsupported system page size` | Image not compatible with 64K-page aarch64 (Qdrant, some TEI versions) | Use a different image — see [gotchas.md](gotchas.md) |
| `ImagePullBackOff` with `no match for platform in manifest` | Image has no `linux/arm64` variant | Use a different image or pin to a known multi-arch tag |
| LiteLLM gateway healthy, but all chat requests time out | One or both 32B replicas are down; gateway tries to retry across both | `kubectl get pods -l app=qwen3-32b-thinking`; restart whichever is wedged |
| `rag-search` returns 502 `embedder unreachable` | embeddings-ollama is restarting or model still loading | `kubectl logs deploy/embeddings-ollama --tail=30` — wait for `bge-m3` to finish loading |
| Wiki ingester job exits 1 but data still landed | One transient 502 from rag-search during the run; rest succeeded | Idempotent — manual re-trigger picks up where it left off |

## Backups directory layout (on the operator workstation, not in git)

A snapshot of the pre-migration cluster state lives at
`backups/2026-05-19-pre-reasoning-migration/` on the operator's local
machine. It's not committed because it contains internal IPs,
PVC names, and pod metadata. If you need to recreate that snapshot
flow, run:

```bash
kubectl get all,configmap,secret,pvc -n lab-domains-sre -o yaml \
  > backup-$(date +%F).yaml
```

…before any destructive change.

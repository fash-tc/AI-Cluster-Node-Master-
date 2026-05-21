# Deployment manifests

One directory per Kubernetes service. Each directory is a self-contained
Kustomize bundle — apply with `kubectl apply -k <directory>`.

## Apply order (cold cluster)

If you're standing up the cluster from scratch, apply in this order so
each layer's dependencies are already present:

1. `rag-stack` — pgvector + embeddings-ollama (no dependencies)
2. `rag-search` — depends on rag-stack
3. `qwen3-32b-thinking-aicompute02` and `qwen3-32b-thinking-aicompute03`
   (independent of each other)
4. `qwen3-235b-thinking`
5. `litellm-gateway` — depends on the vLLM models being ready
6. `ollama-shim` — depends on litellm-gateway and embeddings-ollama
7. `wiki-ingester` — depends on rag-search; needs a Secret created
   out-of-band (see its README)

Every bundle ships with **`replicas: 0`**. Apply creates the
resources but doesn't start anything. Scale up explicitly when ready:

```bash
kubectl -n lab-domains-sre scale deploy/<name> --replicas=1
```

## What's in each bundle

| Bundle | Component | Doc |
|---|---|---|
| `qwen3-235b-thinking/` | Qwen3-235B-A22B reasoning model | [docs/components/qwen3-235b-thinking.md](../docs/components/qwen3-235b-thinking.md) |
| `qwen3-32b-thinking-aicompute02/` | Qwen3-32B replica A | [docs/components/qwen3-32b-thinking.md](../docs/components/qwen3-32b-thinking.md) |
| `qwen3-32b-thinking-aicompute03/` | Qwen3-32B replica B | [docs/components/qwen3-32b-thinking.md](../docs/components/qwen3-32b-thinking.md) |
| `litellm-gateway/` | OpenAI-compatible front door | [docs/components/litellm-gateway.md](../docs/components/litellm-gateway.md) |
| `ollama-shim/` | Ollama-API translation layer | [docs/components/ollama-shim.md](../docs/components/ollama-shim.md) |
| `rag-stack/` | pgvector + embeddings-ollama (combined) | [docs/components/pgvector.md](../docs/components/pgvector.md), [docs/components/embeddings-ollama.md](../docs/components/embeddings-ollama.md) |
| `rag-search/` | REST wrapper over pgvector + embeddings | [docs/components/rag-search.md](../docs/components/rag-search.md) |
| `wiki-ingester/` | Daily Confluence → RAG sync | [docs/components/wiki-ingester.md](../docs/components/wiki-ingester.md) |

## Secrets that must be created out-of-band

Two bundles contain placeholder secrets that must be replaced with real
values before applying. Both intentionally avoid committing real
credentials to git.

### `rag-stack/pgvector-secret.yaml`

Placeholder `POSTGRES_PASSWORD: CHANGE_ME_BEFORE_APPLY`. Edit before
applying — or create the secret directly:

```bash
kubectl -n lab-domains-sre create secret generic pgvector-credentials \
  --from-literal=POSTGRES_USER=rag \
  --from-literal=POSTGRES_PASSWORD='<real-password>' \
  --from-literal=POSTGRES_DB=rag
```

…and remove `pgvector-secret.yaml` from `rag-stack/kustomization.yaml`'s
`resources:` list before applying the rest.

### `rag-search/secret.yaml`

DSN with the same `CHANGE_ME_BEFORE_APPLY` password. Must match
whatever password is in `pgvector-credentials`. Edit before applying or
create out-of-band:

```bash
kubectl -n lab-domains-sre create secret generic rag-search-pg-dsn \
  --from-literal=dsn='postgresql://rag:<real-password>@domains-pgvector.lab-domains-sre.svc.cluster.local:5432/rag'
```

### `wiki-ingester/secret.yaml`

Placeholder `ATLASSIAN_EMAIL` and `ATLASSIAN_API_TOKEN`.
**The kustomization.yaml in this bundle does NOT include `secret.yaml`**
— the file is a template only. Create the secret out-of-band:

```bash
kubectl -n lab-domains-sre create secret generic wiki-ingester-creds \
  --from-literal=ATLASSIAN_EMAIL='your.email@example.com' \
  --from-literal=ATLASSIAN_API_TOKEN='<atlassian-api-token>'
```

Token requirements documented in
[docs/components/wiki-ingester.md](../docs/components/wiki-ingester.md#credentials).

## Pre-apply dry-run

Always validate before applying for real:

```bash
kubectl apply -k deploy/<bundle> --dry-run=server
```

Catches namespace problems, conflicting resource names, and most
schema errors before they hit the cluster.

## Conventions used across all bundles

- **Namespace:** `lab-domains-sre`. Set via `kustomization.yaml`, not
  per-resource — if you copy a bundle into a different namespace, edit
  the `namespace:` field in `kustomization.yaml`.
- **Service names:** `domains-<name>` for in-cluster discoverable
  services (`domains-llm-gateway`, `domains-pgvector`, etc.).
- **NodePort assignments:**
  - 31434 — ollama-shim
  - 31440 — litellm-gateway
  - 31441 — qwen3-235b direct
  - 31442 — qwen3-32b aicompute02 direct
  - 31443 — qwen3-32b aicompute03 direct
  - 31445 — rag-search
- **Resource requests/limits:** every bundle sets both. Limits are
  generous; requests are conservative to allow co-tenancy of CPU-only
  workloads on the GPU nodes.
- **Startup probes:** model deployments use long startupProbe windows
  (10-min+ failure threshold) because vLLM cold-start time is dominated
  by model loading, not container readiness.
- **`replicas: 0` default:** every model deployment starts paused. Scale
  up explicitly when ready.

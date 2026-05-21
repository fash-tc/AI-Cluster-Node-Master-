# Gotchas

Problems discovered the hard way. Read this before re-treading.

## 64K-page kernel breaks some images

GH200 hosts run a 64 KiB page-size aarch64 kernel
(`6.8.0-1032-nvidia-64k`). Container images that ship native code
compiled for 4K pages crash during memory-allocator startup. Symptom:

```
<jemalloc>: Unsupported system page size
<jemalloc>: Unsupported system page size
memory allocation of 144 bytes failed
Aborted (core dumped)
```

**Known-bad on this cluster:**

- `qdrant/qdrant:v1.13.0` and `qdrant/qdrant:latest` — bundled jemalloc
  hardcoded to 4K. `MALLOC_CONF=narenas:1,tcache:false` workaround had
  no effect.

**Known-good substitute:** `pgvector/pgvector:pg17` — PostgreSQL handles
64K pages cleanly on aarch64. Slightly more code (SQL instead of REST)
but reliable.

Before pulling a new image, check both that an `aarch64` build exists
**and** that someone else has confirmed it works on aarch64-64K. If
nobody has, the safest bet is a Python service that uses well-tested
ARM wheels (`psycopg-binary`, etc.) or pre-built distro packages.

## Some images have no aarch64 build at all

The error here is different:

```
failed to pull and unpack image "X": no match for platform in manifest: not found
```

**Known-bad:** `ghcr.io/huggingface/text-embeddings-inference:cpu-1.5`
and `:cpu-latest`. Neither has a `linux/arm64` variant in the manifest.

**Known-good substitute:** Ollama serving `bge-m3`. The `ollama/ollama`
image is multi-arch and was already proven on this cluster.

## NCCL falls back to TCP cross-node

NCCL diagnostics on this cluster (running the standard
`nccl_all_reduce` test across two nodes) showed pods fall back to
**socket transport** rather than RDMA/GDR. This makes cross-node tensor
parallelism slow and brittle.

**Implication:** every model in this cluster is **single-node**. Big
models that don't fit on a single GPU use Grace memory via
`--cpu-offload-gb`, not GPU-to-GPU sharding across nodes.

A 480B coding model deployment that ignored this rule (Ollama-based,
on `aicompute03` only — not technically cross-node but stressed the
node hard enough) crashed the node to `NotReady` mid-smoke-test. That's
why every new model deployment now starts paused.

## vLLM puts answers in `reasoning_content`

When vLLM is started with `--reasoning-parser deepseek_r1`, it extracts
`<think>...</think>` blocks from the model output into a separate
`message.reasoning_content` field, leaving the final answer in
`message.content`.

**Quirk:** with `chat_template_kwargs.enable_thinking: false`, some
vLLM versions still pipe the whole response through the reasoning
parser, ending up with `content: null` and the actual answer in
`reasoning_content`. OpenAI SDK users see empty `content`.

**Workarounds:**

- The Ollama shim already handles this: it coalesces `content ??
  reasoning_content` before returning.
- For raw OpenAI calls, treat `reasoning_content` as a fallback:
  `text = resp.choices[0].message.content or resp.choices[0].message.reasoning_content`.

## Qwen3 native context is 40K, not 131K

The first vLLM startup of the 32B pair set `--max-model-len 131072` and
crashed with:

```
ValidationError: User-specified max_model_len (131072) is greater than the
derived max_model_len (max_position_embeddings=40960.0 ... )
```

Qwen3-32B's native context is 40960 tokens. Going higher requires YaRN
rope scaling (`--rope-scaling '{"rope_type":"yarn",...}'`), which adds
config surface and accuracy risk. For SRE workloads, 32K is comfortable
— alerts + relevant log windows fit easily.

If you genuinely need 100K+ context, add the rope-scaling config and
test it on representative inputs before relying on it.

## `kubectl apply -k` resets replicas

The Kustomize bundles ship with `replicas: 0`. If you've already scaled
a model to 1 and then re-apply the bundle (e.g. to pick up a configmap
change), apply will silently scale it back to 0. The pod goes away.

**Workaround:** after every `apply -k`, re-scale to 1 if you intended
that:

```bash
kubectl apply -k deploy/<bundle>
kubectl -n lab-domains-sre scale deploy/<name> --replicas=1
```

Alternative: temporarily edit `deployment.yaml` to `replicas: 1` before
applying, then revert. Not recommended — defeats the purpose of the
paused-by-default discipline.

## Atlassian API token must have Confluence scope

Atlassian's newer "scoped" API tokens (format starts with `ATATT3...`)
can be created with product-specific scopes. A Jira-only token gets a
403 from Confluence APIs even though it authenticates fine to Jira:

```
{"statusCode":403,"message":"... Request rejected because caller cannot access Confluence"}
```

When creating a token at
[id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens),
make sure the scopes include Confluence read access. The "classic"
unscoped token format has all the user's permissions automatically.

## Atlassian email differs by product

User `fash@tucowsinc.com` may exist in Jira; the same person in
Confluence is `fash@tucows.com`. The Atlassian Basic Auth pair is
`email:token` — using the wrong email gives a 403, not a 401.

When the API returns "caller cannot access Confluence" with what looks
like a valid token, try alternate email aliases for the same user
before assuming the token itself is wrong.

## local-path PVCs are node-pinned

PVCs use the `local-path` storage class. The PVC binds to a PV on
whatever node first scheduled the consumer pod. After that:

- The pod cannot move nodes without recreating the PVC (and losing the
  data).
- Pod restarts on the same node keep the data.
- Deleting the PVC destroys the data; there is no snapshot.

This is fine for ephemeral caches (model weights — re-pullable) and
acceptable for single-node services (pgvector — but back it up). It's
a real problem if you ever want multi-node redundancy of a stateful
service. Plan for that explicitly if it matters.

## The Ollama shim drops streaming

The shim hard-codes `stream: false` when translating Ollama `/api/chat`
to OpenAI `/v1/chat/completions`. Clients that pass `"stream": true` to
the shim still get a single complete JSON response.

If you need streaming, use the OpenAI gateway at `:31440/v1` directly.

## LiteLLM gateway returns 200 even when backends are unreachable at startup

LiteLLM's `/health/readiness` returns 200 once the proxy is itself
listening. It does **not** verify that any backend in `model_list`
is reachable. If you scale the gateway up before the model deployments
are ready, the gateway will look healthy but every chat completion will
fail with a 502 or timeout.

**Pattern:** scale models first, wait for `kubectl wait
--for=condition=available`, then scale the gateway.

## Wiki-ingester "Failed" status can be misleading

The wiki-ingester job exits with code 1 if **any** page fails
ingestion. A single transient 502 from rag-search (when bge-m3 is busy)
causes the whole job to be marked Failed even though 22 of 23 pages
landed successfully.

When the job is Failed, before assuming the worst:

```bash
# Did data make it in?
kubectl -n lab-domains-sre exec deploy/pgvector -- \
  psql -U rag -d rag -c "SELECT COUNT(*) FROM rag_occ_wiki;"
```

If the count looks right, the job did its job — just re-trigger
manually to pick up whichever 1–2 pages bounced.

## Don't trust the listing API's `body` field

Atlassian's Confluence v2 `GET /api/v2/spaces/{id}/pages` returns
`body: ""` for many pages even when the page has content. To get the
real body, you must request `body-format=storage` (or markdown via the
MCP tool path).

The wiki-ingester correctly passes `body-format=storage` in its list
calls, so this isn't a bug today — but if you ever write a quick script
against the Atlassian API, don't trust `body` from the list endpoint
without also fetching with an explicit body-format.

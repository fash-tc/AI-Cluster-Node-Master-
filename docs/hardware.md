# Hardware

Three NVIDIA GH200 Grace-Hopper Superchip nodes plus three control-plane
nodes. The compute nodes are where every model and service lives.

## Compute nodes

| Node | CPU cores | Grace RAM (CPU-side) | GPU |
|---|---|---|---|
| `aicompute01.cnco1.tucows.cloud` | 72 (NVIDIA Grace, ARM aarch64) | ~600 GiB LPDDR5X | 1 × H100 (96 GB HBM3) |
| `aicompute02.cnco1.tucows.cloud` | 72 | ~600 GiB | 1 × H100 (96 GB HBM3) |
| `aicompute03.cnco1.tucows.cloud` | 72 | ~500 GiB | 1 × H100 (96 GB HBM3) |

Internal IPs are `10.169.25.11/12/13`. All three are `Ready` and have
been since cluster bring-up. Kubernetes versions match across the
cluster: client 1.30, server 1.32.5, runtime containerd 2.0.5.

## What a GH200 is, briefly

A single chip with two halves:

- **Hopper GPU** — H100-class compute with 96 GB HBM3 (high-bandwidth GPU
  memory, ~3 TB/s)
- **Grace CPU** — 72-core ARM Neoverse V2 with ~480 GB LPDDR5X (slower
  per-byte than HBM, but vastly bigger)

The two halves are bridged by **NVLink-C2C** at ~450 GB/s — much faster
than PCIe (~64 GB/s) but slower than HBM3 internal bandwidth. This is
the key fact that makes large MoE models like Qwen3-235B viable on a
single physical GPU: the hot experts live in HBM3, cold experts ride
Grace LPDDR5X, and NVLink-C2C is fast enough that the offload doesn't
fall apart performance-wise.

## GPU sharing model

Kubernetes' device plugin exposes each physical GH200 as **4 time-sliced
shares** (`nvidia.com/gpu: 4`). These are not 4 separate GPUs and not
4 independent memory pools — they are time-slices of one chip with one
HBM3 pool.

In practice, every model deployment in this cluster reserves all 4
shares (`requests: nvidia.com/gpu: 4, limits: 4`) and pins to a single
node via `nodeSelector`. That gives the model the whole physical GPU
without time-slicing overhead and without surprise contention.

The "4 shares" abstraction only matters if you want to colocate two
small GPU workloads on one node, which we don't.

## Why we don't do cross-node tensor parallelism

NCCL diagnostics on this cluster confirmed pods fall back to TCP
sockets when communicating cross-node. That means cross-node tensor
parallelism (split a single big model across multiple GPUs) is slow
enough to be unusable for interactive inference.

Concretely, the first deployment attempts on this cluster tried that
shape (`deepseek-v4-flash-vllm` with pipeline-parallel=3 across all
three nodes) and were paused because of the resulting performance and
fragility.

The current design avoids cross-node communication entirely:

- Each model is **single-node**.
- Big models that don't fit on a single GPU live across HBM3 + Grace via
  CPU-offload (`--cpu-offload-gb N` in vLLM).
- "HA" means **replicas of the same model on different nodes** behind a
  gateway, not splitting one model.

If RDMA/GDR becomes available later (Mellanox/CX-7 + appropriate fabric
config), distributed serving becomes viable. Until then, single-node is
the only thing worth doing.

## 64K-page kernel — what works and what doesn't

GH200 hosts run kernel `6.8.0-1032-nvidia-64k` — an Ubuntu/NVIDIA kernel
built for 64 KiB page size (vs. the typical 4 KiB on x86 Linux).

Software that ships pre-compiled native code is sometimes hardcoded for
4K pages. When that happens, the program aborts during memory
allocator startup. We hit it twice during initial cluster setup:

| Image | Symptom | Status |
|---|---|---|
| `qdrant/qdrant:v1.13.0` and `:latest` | `jemalloc: Unsupported system page size` then SIGABRT | Replaced with **pgvector** (PostgreSQL handles 64K pages cleanly on aarch64) |
| `ghcr.io/huggingface/text-embeddings-inference:cpu-1.5` and `:cpu-latest` | No `linux/arm64` variant in manifest at all | Replaced with **Ollama serving bge-m3** (proven to work on this cluster) |

The general rule: when picking a container image, prefer ones whose
upstream builds against aarch64 and tests on Grace-class kernels.
Python-based services with no native deps (or only widely-supported
ones like `psycopg-binary`, `html2text`, `urllib`) work without issue.

## Storage

Per-pod persistent storage uses the `local-path` provisioner. Important
implications:

- **PVCs are pinned to whichever node first scheduled the consuming
  pod.** When the pod is rescheduled to a different node, the PVC stays
  bound to the original node; the pod will not start until it lands
  back on that node. For single-node workloads this is fine.
- **Deleting a PVC destroys its data.** There is no replication, no
  snapshot. The model caches are re-pullable from Hugging Face. The
  pgvector database is **not** re-derivable — back it up before any
  destructive operation.
- **No cluster-scoped read access** to the underlying PVs is granted to
  the operator user (`fash`). PV/StorageClass-level inspection requires
  cluster-admin.

Approximate per-PVC sizes:

| PVC | Size | Notes |
|---|---|---|
| `qwen3-235b-thinking-cache` | 320 GiB | Hugging Face cache for the 235B FP8 weights (~220 GiB + headroom) |
| `qwen3-32b-thinking-aicompute02-cache` / `-03-cache` | 80 GiB each | 32B FP8 weights (~32 GiB) + headroom |
| `pgvector-data` | 50 GiB | PostgreSQL data dir (vectors + metadata) |
| `embeddings-ollama-data` | 20 GiB | Ollama model dir (`bge-m3` is ~1.2 GiB) |
| `rag-search-pydeps` | 1 GiB | Pip cache for `psycopg-binary` |
| `wiki-ingester-pydeps` | 256 MiB | Pip cache for `html2text` |

## Cluster access

The kubeconfig points to `https://aik8.cnco1.tucows.cloud:6443`. Auth is
**OpenStack-token-based** via Keystone — the kubeconfig's `exec` block
shells out to bash and reads `$OS_TOKEN` from the environment.

To get a token:

```powershell
$env:OS_AUTH_URL='https://openstack.cnco.tucows.cloud:5000'
$env:OS_PROJECT_NAME='prod-ai-k8-rbac'
$env:OS_USER_DOMAIN_NAME='TUCOWS'
$env:OS_USERNAME='<your-username>'
$env:OS_PASSWORD='<your-openstack-password>'
$env:OS_REGION_NAME='cnco2'
$env:OS_INTERFACE='public'
$env:OS_IDENTITY_API_VERSION='3'

$env:OS_TOKEN = & "$env:APPDATA\Python\Python313\Scripts\openstack.exe" token issue -f value -c id
```

Tokens are valid for ~24 hours. Re-issue daily.

The user's permissions are namespace-scoped to `lab-domains-sre` and do
not include cluster-scoped resources (PVs, StorageClasses, DaemonSets).
For changes that need those, escalate to cluster-admin.

## Out-of-scope but worth knowing

- **No GPU monitoring stack.** There is no Prometheus, no DCGM exporter,
  no node-exporter. Health checks rely on `kubectl get nodes` plus the
  vLLM serving metrics each model exposes on its own port. Adding
  observability is a known follow-up.
- **No backup automation.** PVCs are local-path and not snapshotted. If
  pgvector data starts mattering long-term, run a `pg_dump` cron and
  ship the output to durable storage.
- **No autoscaling.** Workloads are sized for the cluster, not the
  reverse. Manual `kubectl scale` is the only knob.

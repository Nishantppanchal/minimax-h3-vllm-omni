# 04 – Kubernetes

The manifests in [`k8s/`](../k8s) run one H3 server per pod on a single node with 2 GPUs.

## Prerequisites

- NVIDIA GPU Operator or device plugin; both GPUs on the **same node**.
- GPU driver supporting CUDA 13 (`nvidia-smi`).
- Hugging Face access to `MiniMaxAI/MiniMax-H3` (gated: accept the license first) and a
  token stored as a secret, if you download in-cluster.
- A fast storage class (local NVMe or high-throughput network storage).

## Apply

```bash
kubectl create namespace minimax-h3
kubectl -n minimax-h3 create secret generic hf-token --from-literal=token=hf_xxx

# Edit storageClassName and the GPU nodeSelector first:
#   kubectl get nodes -L nvidia.com/gpu.product
kubectl apply -f k8s/minimax-h3.yaml

# Only if the weights are not already on the volume:
kubectl apply -f k8s/minimax-h3-download-job.yaml
kubectl -n minimax-h3 logs -f job/minimax-h3-download

kubectl -n minimax-h3 logs -f deploy/minimax-h3
```

The server pod may restart while the download runs; that's expected.

If your weights already live elsewhere (another PVC, hostPath, NFS), point the `weights`
volume at it and skip the Job. The server needs the folder containing `model_index.json`
and `FL2VA/` (plus `Ref2VA/` for combined mode).

## Design choices

| Setting | Reason |
|---|---|
| `replicas: 1`, `strategy: Recreate` | A rolling update would need 4 free GPUs at once |
| `/dev/shm` as `emptyDir: {medium: Memory}` | Equivalent of Docker `--ipc=host`; vLLM needs shared memory for tensor parallelism |
| Long `startupProbe` (up to 40 min) | Weight loading and compile take many minutes |
| `/health` for readiness and liveness | Same as vLLM's own Kubernetes guide |
| `enableServiceLinks: false` | Prevents injected service env vars colliding with vLLM's |
| `HF_HUB_OFFLINE=1` | Weights are local; never download at startup |
| Memory 64 GiB request / 160 GiB limit | Resident path estimate; measure and tune ([02](02-hardware-and-sizing.md)) |
| CPU request, no CPU limit | MP4 encoding is CPU; limits cause throttling |

## Networking

- Generation takes minutes. Use the **async** `POST /v1/videos` API, or raise ingress
  read timeouts to ~3600 s for `/v1/videos/sync`.
- Raise ingress body size if clients upload reference images or videos (≤ 64 MB total).
- With multiple replicas behind a gateway, async job IDs must route back to the pod that
  owns the job (sticky routing). Plain round-robin breaks polling.

## Scaling

One server processes one generation at a time ([05](05-performance-and-concurrency.md)).
To serve more users, add replicas (one GPU pair each) behind a job queue, and autoscale
on queue depth rather than CPU or GPU utilisation.

## Mixed workloads

Run separate Deployments for:
- normal H3 vs Turbo LoRA (switching costs ~24 s per request), and
- FL2VA vs Ref2VA on 2× H100 (both don't fit on one pair).

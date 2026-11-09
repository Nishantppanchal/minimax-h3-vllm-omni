# MiniMax-H3 on vLLM-Omni: Deployment Notes

Practical notes for self-hosting the **MiniMax-H3** audio-video generation model with
**vLLM-Omni** on Kubernetes, on **2× H100 80 GB** or **2× H200 141 GB**, plus a comparison
with **LTX-2.5** as a faster alternative.

> **Status:** compiled 2026-11-10 from the vLLM-Omni release notes, the official H3 recipe
> (vllm-omni `main`), recipes.vllm.ai, Hugging Face model cards and community benchmarks.
> This stack changes weekly. Re-check versions and flags before relying on anything here.
> Numbers marked *estimate* are extrapolations, not measurements.

> **License warning:** the MiniMax-H3 community license excludes some territories
> (reported: US, EU, UK, South Korea) and may cap free commercial use by revenue.
> Read the license and get legal review before any enterprise deployment.

## TL;DR

| Question | Answer |
|---|---|
| Container | `vllm/vllm-omni:v0.32.0` (current stable, = `latest`) |
| Min. vLLM-Omni for H3 | 0.26 (first support); use **≥ 0.28**, ideally 0.32 |
| GPUs | 2× H100 works for FL2VA-only; 2× H200 is far more comfortable |
| Weights actually loaded | ~135 GB (FL2VA only), ~200 GB (+Ref2VA); full HF repo is ~465 GB |
| Concurrency | One generation at a time per server; scale with replicas |
| Speed (2× H200, 5 s clip, 768p) | ~3 min full quality (*estimate*) |
| Fastest alternative | LTX-2.5 distilled: ~3.4 s (960×544) / ~10.6 s (1080p) per 5 s clip on 2× H200 (measured) |

## Contents

| Doc | Topic |
|---|---|
| [01 – Versions and container](docs/01-versions-and-container.md) | Which vLLM-Omni release, image tags, CUDA/driver |
| [02 – Hardware and sizing](docs/02-hardware-and-sizing.md) | GPU memory, host RAM, CPU, disk, H100 vs H200 |
| [03 – Serving and API](docs/03-serving-and-api.md) | `vllm serve` flags, request fields and rules |
| [04 – Kubernetes](docs/04-kubernetes.md) | How the manifests work and how to apply them |
| [05 – Performance and concurrency](docs/05-performance-and-concurrency.md) | Why it is slow, why it can't batch like an LLM, tuning |
| [06 – Turbo LoRA](docs/06-turbo-lora.md) | 4-step distilled adapter: speed, setup, rules |
| [07 – Video editing](docs/07-video-editing.md) | Ref2VA editing and latent-mask editing |
| [08 – Model comparison (LTX-2.5)](docs/08-model-comparison.md) | Size, speed, editing, license |
| [09 – What works / what doesn't](docs/09-status-matrix.md) | Confirmed, untested, known limits |
| [10 – Sources](docs/10-sources.md) | Where each fact came from |

## Manifests

| File | Purpose |
|---|---|
| [`k8s/minimax-h3.yaml`](k8s/minimax-h3.yaml) | PVC, Deployment, Service for H3 |
| [`k8s/minimax-h3-download-job.yaml`](k8s/minimax-h3-download-job.yaml) | Optional: fetch H3 weights into the PVC |
| [`k8s/ltx-2.5.yaml`](k8s/ltx-2.5.yaml) | PVC, Deployment, Service for LTX-2.5 (bake-off) |
| [`k8s/ltx-2.5-download-job.yaml`](k8s/ltx-2.5-download-job.yaml) | Optional: fetch LTX-2.5 weights |

## Quick start (H3)

```bash
kubectl create namespace minimax-h3
kubectl -n minimax-h3 create secret generic hf-token --from-literal=token=hf_xxx
# edit storageClassName + nodeSelector first
kubectl apply -f k8s/minimax-h3.yaml
kubectl apply -f k8s/minimax-h3-download-job.yaml   # skip if weights are already on the volume
kubectl -n minimax-h3 logs -f deploy/minimax-h3     # wait for "Application startup complete"
```

Then follow the smoke test in [03 – Serving and API](docs/03-serving-and-api.md).

## Disclaimer

Community notes, not affiliated with MiniMax, Lightricks or the vLLM project.
Verify against upstream documentation before production use.

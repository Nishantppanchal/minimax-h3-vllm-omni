# 01 – Versions and container

## Use vLLM-Omni, not plain vLLM

MiniMax-H3 is a diffusion model that generates video **and** audio. It is served by
**vLLM-Omni** (`vllm serve ... --omni`), which extends vLLM to non-text modalities.

## Which version

vLLM-Omni publishes a stable release aligned with every even-numbered vLLM minor version.
The container bundles the matching vLLM, so you never install vLLM separately.

| vLLM-Omni | H3 status | Recommendation |
|---|---|---|
| ≤ 0.24 | No H3 | No |
| 0.26.0 | First H3 support (T2VA, FL2VA, Ref2VA) | Too early: missing the memory features |
| 0.28.0 | Modular pipelines, global FP8 (DiT + text encoder), DLO, text-encoder disaggregation, TeaCache/Cache-DiT | Minimum to use |
| 0.30.0 | Component-selective offload, complete VSA/Ulysses support, full Turbo LoRA matrix | Good |
| **0.32.0** | Current stable (as of 2026-11-10); latent-mask editing, more Turbo and continuation work | **Recommended** |
| 0.33.0rc1 | Pre-release (2026-11-04); Docker image not yet published | Wait for 0.33.0 final |

## Container image

```
vllm/vllm-omni:v0.32.0
# pinned (linux/amd64):
vllm/vllm-omni:v0.32.0@sha256:6b44ee504a0dc3a21eacf72e7dc5da318b7bcc8ed0e349dc928e67278414430c
```

- `latest` currently points to v0.32.0, but it moves. Pin a tag or digest in production.
- The CUDA image has **no default entrypoint**: always pass `vllm serve ... --omni`.
- Built for **CUDA 13**. Check that `nvidia-smi` on GPU nodes shows CUDA 13.0 or newer.
- Verify ffmpeg is present (needed for MP4 output and reference-video preprocessing):

```bash
docker run --rm vllm/vllm-omni:v0.32.0 which ffmpeg ffprobe
```

If either is missing, build a thin image on top that installs ffmpeg.

## Outdated advice you may still find

| Advice | Status |
|---|---|
| Use `vllm/vllm-omni:minimax-h3` and mount a vllm-omni source checkout on `PYTHONPATH` | Workaround from before the modular H3 pipeline was released. Not needed on ≥ 0.28 |
| `HF_HUB_ENABLE_HF_TRANSFER=1` for downloads | `hf_transfer` is gone in huggingface_hub 1.x; use `HF_XET_HIGH_PERFORMANCE=1` |
| "Online FP8 is incompatible with layerwise offload / quantizes the DiT only" (recipes.vllm.ai, Oct 21) | Superseded: global FP8 covers DiT + text encoder and works with DLO (0.28+) |

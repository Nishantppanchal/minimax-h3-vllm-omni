# 05 – Performance and concurrency

## Published timings

| Hardware | Clip | Time | Source |
|---|---|---|---|
| 2× H100, TP2, BF16 | ~8.7 s at 672×384, 30 steps | ~45 s each (4 requests in 174.8 s) | H3 recipe |
| 2× H100, TP2, online INT8 | same | 153.3 s for 4 (vs 174.8 s BF16), 56.9 GB vs 72.4 GB peak | H3 recipe |
| 2× H200, TP2, BF16 | 4 s at 512×288, 16 steps | ~5 s | Intel model card |
| 4× H200 | 5 s at 1344×768, 50 steps | 85.49 s median | H3 recipe |
| 4× B300 | 8.7 s FL2VA at 1248×768 | 86.96 s | H3 recipe |
| 4× MI300X | 8.7 s T2VA at 1344×768 | 267.42 s | H3 recipe |
| 4× B300 | 15 s two-video Ref2VA edit | 784.4 s | H3 recipe |
| 8× B300 | 15 s FL2VA at 1344×768 | 112.5–112.9 s end to end | H3 recipe |
| 8× B300 + Turbo v1.2 | same | ~16.5 s | H3 recipe |

## Estimates for 2 GPUs (*not measured*)

| Clip | 2× H100 | 2× H200 |
|---|---|---|
| 5 s, 1344×768, 50 steps | ~3 min | ~3 min |
| 15 s, 1344×768, 50 steps | ~10–20 min (likely needs FP8 on H100) | ~10–20 min |

Rule of thumb: ~20 full-quality 5 s 768p clips per hour per GPU pair.

## Why it's slow on 2 Hopper GPUs

1. **Less compute than the benchmarks.** 8× B300 is very roughly 16–20 H100s of compute.
2. **Huge sequences.** Tokens per step ≈ (W/32) × (H/32) × latent frames. 672×384 at
   ~9 s is ~16k tokens; 1344×768 at 15 s is ~108k (*my calculation*). Attention grows with
   the square of this, so long or high-resolution clips get disproportionately expensive.
3. **Attention dominates** (~76% of denoise time in one profile). Blackwell uses
   TRTLLM/FlashAttention-4; Hopper defaults to FlashAttention 2.
4. **Settings left on the table:** BF16 instead of FP8, `--enforce-eager`, and TP traffic
   over PCIe if the GPUs lack NVLink.

H200 doesn't fix this: same compute as H100, only more memory and bandwidth.

## Why requests can't be batched like an LLM

- **LLM decode is memory-bandwidth bound:** one token per user per step, so the GPU waits
  on weight reads. Batching many users reuses one weight read, nearly free.
- **Video diffusion is compute bound:** every step processes the whole video (tens of
  thousands of tokens), so the math units are already saturated. Batching N requests
  costs N× the FLOPs.

Measured on 2× H100 with 4 simultaneous requests (672×384, 30 steps):

| Mode | Total | Avg latency |
|---|---|---|
| One at a time | 174.8 s | 111.5 s |
| Co-batched (4 together) | 182.1 s | 175.7 s |

Per-request denoise cost fell only 2.4% (1.323 → 1.291 s/step). Keep `--max-num-seqs 1`.

Continuous batching and KV caching don't transfer: there is no cheap autoregressive
phase, and every step changes the whole latent.

## Running requests in parallel anyway

All of these trade latency for concurrency; total videos per hour stays about the same.

| Option | Notes |
|---|---|
| `--step-execution` | New requests start at the next step boundary (~1.3 s) instead of waiting ~45 s. Same throughput |
| One server per GPU | 2 videos at once, each ~2× slower. Feasible on H200 with FP8; H100 needs CPU offload |
| DLO data parallel (`--enable-distributed-layerwise-offload --data-parallel-size 2`, AllGather) | One server, 2 concurrent requests. Needs ≥ 200 GiB host RAM; no H100 numbers |

## What actually raises throughput

| Lever | Gain |
|---|---|
| More GPU pairs (replicas) | Linear |
| Turbo LoRA | ~7× end to end (8× B300) |
| `quality=high` per request (Cache-DiT) | 1.35× on 4× H200, SSIM 0.971 vs lossless |
| FP8 / INT8 | ~12% on 2× H100 (INT8 measured), plus memory headroom |
| Remove `--enforce-eager` | ~5–10% |
| Shorter / lower-resolution clips | Largest single lever |
| Faster model (LTX-2.5) | See [08](08-model-comparison.md) |

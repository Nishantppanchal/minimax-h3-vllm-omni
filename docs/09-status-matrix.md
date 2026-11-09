# 09 – What works, what's untested, what doesn't

As of 2026-11-10, vLLM-Omni 0.32.0.

## ✅ Confirmed

- `vllm/vllm-omni:v0.32.0` image (CUDA 13), `latest` points to it.
- 2× H100, TP2, BF16, FL2VA only at 672×384 (peak 72.4 GB/GPU). Tracked by maintainers as
  a baseline (text-encoder TP2 restored after a startup OOM regression).
- Global FP8 (DiT + text encoder): one TP2 test 68.52 → 53.51 GiB per GPU. Combines with DLO.
- Online INT8 on 2× H100: ~12% faster, 72.4 → 56.9 GB.
- T2VA and FL2VA from the FL2VA partition (`--task-type fl2va`).
- Turbo LoRA (full LightX2V matrix since 0.29/0.30), Ref2VA editing, latent-mask editing.
- `/dev/shm` volume and `/health` probes, per vLLM's Kubernetes guide.

## ❓ Untested / unpublished

- 2× H100 at 1344×768 in BF16 (likely OOM; plan on FP8).
- 2× H200 combined FL2VA + Ref2VA (both DiTs).
- ffmpeg in the general v0.32.0 image.
- Host RAM for the resident path.
- Best attention backend on Hopper (default FlashAttention 2 vs `CUDNN_ATTN`).
- Turbo + FP8 together.
- DLO data-parallel (2 concurrent requests) throughput on Hopper.
- Which partition latent-mask editing requires.

## ❌ Doesn't work / known limits

- Ref2VA on an FL2VA-only server.
- Batching for throughput (co-batching gives ~2% per-step gain, worse latency).
- `--cfg-parallel-size` > 1 (rejected); Ulysses 2 × Ring 2 hybrid (attention-mask mismatch).
- Full Ref2VA reference combinations (serving accepts a subset).
- Output outside 4–15 s or other than 24 FPS.
- Mixing Turbo and normal requests efficiently on one server (~24 s switch).
- LTX-2.5 IC-LoRA (video editing) in vLLM-Omni.
- `hf_transfer` downloads; the old `minimax-h3` image + source overlay (obsolete).

## ⚖️ Licensing

- H3: community license excludes some territories (reported: US, EU, UK, South Korea);
  free commercial use reportedly capped by revenue. Legal review required.
- LTX-2.5: paid agreement from $10M annual revenue.
- Wan 2.2: Apache 2.0.

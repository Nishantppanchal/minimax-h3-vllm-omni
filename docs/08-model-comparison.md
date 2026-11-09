# 08 – Model comparison

## Open-weight options (Nov 2026)

| Model | Quality | Audio | Editing in vLLM-Omni | License |
|---|---|---|---|---|
| **MiniMax-H3** | Highest-ranked open-weight video model on Artificial Analysis | Yes | Yes (Ref2VA, latent masks) | Community license; excludes some territories (reported US, EU, UK, South Korea); one source reports free use only under $20M revenue with attribution |
| **LTX-2.5** (Lightricks) | Lower than H3 | Yes, up to 4K | No (IC-LoRA unsupported) | Paid commercial agreement from $10M annual revenue; no territory exclusion |
| **Wan 2.2** (Alibaba) | Lower | No native audio | VACE companion models | Apache 2.0, no revenue or territory limits |
| Runway Aleph 2, Gemini Omni Flash | Strong editing | Yes | n/a | API only, no self-hosting |

Higher-ranked models (Seedance, Veo, Wan 3.0) are API-only.

## H3 vs LTX-2.5: size

| | LTX-2.5 | MiniMax-H3 (per task) |
|---|---|---|
| Main model | 22B transformer | 33B DiT (~66 GB BF16) |
| Text encoder | Gemma 4 12B | Qwen3-VL derived (~51.5 GB BF16) |
| Total weights (BF16) | ~70 GB | ~135 GB (FL2VA partition) |
| GPUs per server | 1 (H100 80 GB with FP8 at 960×544; H200 for 1080p) | 2 |
| Steps | 8 (distilled) or 8 + 3 (two-stage) | 50 (4 with Turbo) |

## H3 vs LTX-2.5: speed on 2× H200

LTX numbers are **measured** (third-party benchmark, vLLM-Omni 0.30.0, 121 frames ≈ 5 s).
H3 numbers are *estimates*.

| Setup | ~5 s clip | Clips/hour |
|---|---|---|
| H3 full quality, 768p | ~3 min | ~20 |
| H3 + Turbo, 768p | ~30–40 s | ~100 |
| **LTX-2.5 distilled two-stage, 1920×1088** | **10.6 s** | **~380** |
| **LTX-2.5 distilled one-stage, 960×544** | **3.42 s** | **~1,200** |

Other measured LTX points from the same benchmark: TP2 at 960×544 was 4.06 s (Ulysses
is faster); on 2× H100, TP2 two-stage 1088×1920 was ~11 s.

Long clips: LTX barely slows with length (RTX 4090: ~44 s for 5 s, ~57 s for 15 s,
~68 s for 20 s), while H3 cost grows faster than linearly.

## Running LTX-2.5 on vLLM-Omni

Same container (`vllm/vllm-omni:v0.32.0`), same `/v1/videos` API.

| Pipeline | Default size | Steps |
|---|---|---|
| `LTX2DistilledOneStagePipeline` | 960×544 | 8 |
| `LTX2DistilledTwoStagePipeline` | 1920×1088 | 8 + 3 |
| `LTX2Pipeline` (full) | 960×544 | 30 |
| `LTX2TwoStagePipeline` (full) | 1920×1088 | 30 + 3 |

2× H200, fastest path (Ulysses, degrees matched to GPU count):

```bash
vllm serve Lightricks/LTX-2.5-Diffusers --omni \
  --model-class-name LTX2DistilledTwoStagePipeline \
  --usp 2 --vae-patch-parallel-size 2 --vae-use-tiling
```

1× H100 (official recipe scope):

```bash
vllm serve Lightricks/LTX-2.5-Diffusers --omni \
  --model-class-name LTX2DistilledOneStagePipeline \
  --quantization fp8 --diffusion-attention-backend CUDNN_ATTN
```

Gotchas:
- Two-stage 1920×1088 needs ~114 GB: doesn't fit on one 80 GB H100. Ulysses replicates
  the model per GPU, so on 2× H100 use TP2 for two-stage.
- Distilled pipelines reject negative prompts.
- One-stage width/height divisible by 32; two-stage final size divisible by 64.
- Check the LTX-2.5 recipe for request fields; they differ from H3's.

## Recommendation

1. Resolve the H3 license question first.
2. Pilot H3 (FP8, async API) since the weights are already downloaded.
3. Bake-off: same prompts, same GPUs, LTX-2.5 distilled vs H3 (and H3 Turbo).
4. Users waiting on screen or high volume → LTX-2.5. Submit-and-come-back with quality as
   the priority → H3, scaled with more GPU pairs.

Keep the application on the async `/v1/videos` API so swapping models is a server change,
not an app rewrite.

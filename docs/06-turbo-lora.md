# 06 – Turbo LoRA

A distilled adapter from LightX2V (`lightx2v/Minimax-h3-Turbo`) that cuts the denoising
loop from 49 model passes to 4.

## Speed

Measured on 8× B300, 15 s clip at 1344×768, Turbo v1.2 vs normal 50-step H3:

| Stage | Normal | Turbo | Speedup |
|---|---|---|---|
| Denoising | ~105 s | ~9 s | 11.7× |
| Whole request incl. MP4 | ~112.7 s | ~16.5 s | 6.8× |

On 2× H100/H200 (*estimate*): ~10–15 s per short low-res clip instead of ~45 s, and
~1.5–2 min for a 10 s 768p clip instead of ~6–10 min.

## Size

~2 GB per adapter file (v1.0 768p export: 1,956,192,992 bytes). Budget some extra GPU
memory too: the adapter stays resident, and its in-memory footprint can exceed the file
size depending on the parallel layout.

Use the original Diffusers-layout files from `lightx2v/Minimax-h3-Turbo`. vLLM-Omni refuses
the `_comfyui_` exports, including ~300 MB "resized" community versions.

## Setup

Download one artifact into its own folder (a folder with several artifacts is rejected as
ambiguous):

```bash
hf download lightx2v/Minimax-h3-Turbo \
  minimax_h3_fl2v_turbo_4step_v1.2_768p_bf16.safetensors \
  --local-dir /models/MiniMax-H3/Turbo-LoRA
```

Add to the server command:

```
--task-type fl2va
--lora-backend peft
--lora-path /models/MiniMax-H3/Turbo-LoRA/minimax_h3_fl2v_turbo_4step_v1.2_768p_bf16.safetensors
```

## Requests

Every request must activate the adapter and use the artifact's sampling settings
(v1.2 and v1.0: 5 inference steps, flow shift 6). Mismatches are rejected.

```bash
curl --fail-with-body -sS -X POST http://127.0.0.1:8000/v1/videos/sync \
  -F 'prompt=A cat plays piano in a sunny room, with soft piano music.' \
  -F 'width=672' -F 'height=384' -F 'aspect_ratio=16:9' -F 'fps=24' \
  -F 'num_inference_steps=5' -F 'flow_shift=6' -F 'seed=1101' \
  -F 'extra_params={"task":"t2va","duration":5.0,"audio_flow_shift":3.0}' \
  -F 'lora={"name":"h3-turbo-v1.2","path":"/models/MiniMax-H3/Turbo-LoRA/minimax_h3_fl2v_turbo_4step_v1.2_768p_bf16.safetensors","scale":1.0}' \
  -o turbo.mp4
```

The `path` must be the path inside the container.

## Rules and limits

- **Quality:** distilled, so expect some loss. Compare on your own prompts.
- **Don't mix modes on one server:** switching between Turbo and normal requests costs
  ~24 s (all 208 LoRA layers are torn down and re-uploaded). Run separate Deployments.
- **Ref2VA** needs the `ref2v` artifacts and `--task-type ref2va`.
- **Offload:** DLO works with the LoRA; model-level and standard layerwise offload don't.
- **FP8 + Turbo:** not documented; test before relying on it.
- If startup refuses v1.2, fall back to v1.0 (same request settings).

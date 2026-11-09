# 03 – Serving and API

## Server command (2 GPUs, FL2VA only)

Point the server at the weights root (the folder containing `model_index.json` and `FL2VA/`).

```bash
VLLM_WORKER_MULTIPROC_METHOD=spawn \
VLLM_OMNI_VIDEO_SYNC_TIMEOUT=14400 \
vllm serve /path/to/MiniMax-H3 --omni --trust-remote-code \
  --host 0.0.0.0 --port 8000 \
  --served-model-name MiniMaxAI/MiniMax-H3 \
  --task-type fl2va \
  --num-gpus 2 --tensor-parallel-size 2 --usp 1 --ring 1 \
  --text-encoder-tp-size 2 \
  --vae-patch-parallel-size 2 --vae-parallel-mode tile --vae-use-tiling \
  --enforce-eager
```

| Flag | Why |
|---|---|
| `--task-type fl2va` | Loads only the FL2VA partition (serves T2VA + FL2VA). Remove on 2× H200 to also serve Ref2VA (needs `Ref2VA/` files) |
| `--tensor-parallel-size 2` | Shards the DiT across both GPUs |
| `--text-encoder-tp-size 2` | Shards the text encoder; must divide its 64 attention / 8 KV heads |
| `--vae-patch-parallel-size 2` | Must be 1 or the full DiT group size |
| `--enforce-eager` | Safer first run. Remove later for regional compile (~5–10% faster; first request is a warmup) |

Optional flags:

| Flag | When |
|---|---|
| `--quantization fp8` | OOM on H100 at 1344×768, or ~10–20% speedup on Hopper |
| `--enable-distributed-layerwise-offload --dlo-no-use-allgather --dlo-resident-layers 30` | Still OOM after FP8. Needs ≥ 200 GiB host RAM. Combines with FP8 |
| `--diffusion-attention-backend CUDNN_ATTN` | Try vs Hopper default (FlashAttention 2) and benchmark |
| `--lora-backend peft --lora-path <file>` | Turbo LoRA, see [06](06-turbo-lora.md) |

Startup can take 10+ minutes. Wait for `Application startup complete`; `/health` then returns 200.

## Endpoints

| Endpoint | Use |
|---|---|
| `POST /v1/videos/sync` | Blocks until done, returns the MP4. Simple, but holds the connection for minutes |
| `POST /v1/videos` | Async job; poll for status, then download. **Use this in production** |
| `GET /health` | Liveness/readiness |

## Request fields (multipart form)

| Field | Value |
|---|---|
| `prompt` | Required, non-empty, ≤ 7,000 characters |
| `fps` | `24` (fixed) |
| `num_inference_steps` | `50` (normal model) |
| `flow_shift` | `12` (normal model) |
| `width`, `height` | Multiples of 32; aspect ratio between 1:4 and 4:1 (e.g. 1344×768, 672×384) |
| `aspect_ratio` | T2VA: one of `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16` |
| `seed` | Optional; fix it for reproducible output |
| `extra_params` | JSON, see below |

`extra_params`:

```json
{"task": "t2va", "duration": 5.0, "audio_flow_shift": 3.0}
```

- `task`: `t2va`, `fl2va` or `ref2va`; must be compatible with the server's `--task-type`.
  An FL2VA-only server accepts `t2va` and `fl2va`.
- `duration`: 4–15 seconds.

Per task:

- **FL2VA:** add `input_reference=@frame.png`. Output follows the input image's aspect
  ratio (a generic `aspect_ratio` is ignored). First + last frame: two references with
  `"frame_indices":[0,-1]`.
- **Ref2VA:** needs the Ref2VA partition. The serving path accepts exactly one image +
  one audio reference, or one or more videos (MP4/MOV, 2–15 s each, ≤ 15 s combined,
  max 3) with no separate audio.

Keep the whole request body under 64 MB; prefer URLs over inline uploads for large media.

## Smoke test

```bash
kubectl -n minimax-h3 port-forward svc/minimax-h3 8000

curl --fail-with-body -sS -X POST http://127.0.0.1:8000/v1/videos/sync \
  -F 'prompt=A cat plays piano in a sunny room, with soft piano music.' \
  -F 'width=672' -F 'height=384' -F 'aspect_ratio=16:9' \
  -F 'fps=24' -F 'num_inference_steps=50' -F 'flow_shift=12' \
  -F 'seed=1101' \
  -F 'extra_params={"task":"t2va","duration":5.0,"audio_flow_shift":3.0}' \
  -o test.mp4
```

Small and safe; expect a minute or two. Then try 1344×768.

## Troubleshooting

| Symptom | Fix |
|---|---|
| CUDA out of memory | `--quantization fp8`; then DLO if needed |
| `unrecognized arguments` at startup | Flag not in this release; check the release notes for the image you run |
| Pod OOMKilled during load | Raise the memory limit (page cache + `/dev/shm` count) |
| `which ffmpeg` prints nothing | Build a thin image that installs ffmpeg |
| Ingress 504 on `/v1/videos/sync` | Raise proxy read timeout (~3600 s) or switch to async `/v1/videos` |

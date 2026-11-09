# 02 – Hardware and sizing

## Model components (BF16)

| Component | Size |
|---|---|
| DiT (one per task family: FL2VA, Ref2VA) | 33B params, ~66 GB each |
| Text encoder (Qwen3-VL derived) | ~51.5 GB |
| Video VAE + audio VAE | ~10 GB |

## Weights on disk

The Hugging Face repo (`MiniMaxAI/MiniMax-H3`) is **~465 GB** because it ships the same
weights several times: a root-level modular library (`transformer/`, `transformer_ref/`,
`text_encoder/`, `vae/`, ...) **and** two self-contained pipelines (`FL2VA/`, `Ref2VA/`).

vLLM-Omni only reads:

| Setup | Files | Approx. size |
|---|---|---|
| FL2VA only (T2VA + FL2VA) | `model_index.json`, `FL2VA/` | ~135 GB |
| Combined (+ Ref2VA) | above + `Ref2VA/model_index.json`, `Ref2VA/transformer/` | ~200 GB |
| Full repo | everything | ~465 GB |

You can delete the root-level duplicates to reclaim ~265 GB, if nothing else (diffusers
scripts, ComfyUI) uses them. Check with `du -sh /path/to/MiniMax-H3/*` first.

## GPUs

| | 2× H100 80 GB | 2× H200 141 GB |
|---|---|---|
| FL2VA, TP2, BF16, 672×384 | Works (measured peak 72.4 GB/GPU) | Works |
| 15 s at 1344×768, BF16 | Likely OOM; use `--quantization fp8` | Should fit |
| FL2VA + Ref2VA on one server | No: run separate servers | Likely fits (*untested*) |
| One server per GPU (2 independent servers) | Needs CPU offload (slow, big RAM) | Yes, with FP8 |
| Speed vs the other | Same Hopper compute | ~0–15% faster (more bandwidth); H3 is compute-bound |

H200 doesn't make H3 much faster. It removes nearly all memory problems.

Check GPU interconnect: `nvidia-smi topo -m` should show `NV#` between the pair.
PCIe-only (`PHB`/`SYS`) makes TP2 noticeably slower.

## Host RAM

Depends on whether weights live on GPU (resident) or in host memory (offload).

| Setup | Host RAM |
|---|---|
| Resident (default; FP8 optional) | Start with 64 GiB request / 160 GiB limit, then measure (*no published figure*) |
| DLO or `--enable-cpu-offload` | ≥ 200 GiB free; 384 GiB node recommended |

Notes:
- Startup is the peak: reading ~135 GB of safetensors fills page cache, which counts
  against the cgroup limit (reclaimable, but a tight limit can still OOMKill the pod).
- `/dev/shm` (memory-backed) counts against the pod memory limit.
- MP4 encoder peak RSS measured at ~2.8 GiB.
- FP8 + DLO **AllGather** temporarily materialises the full FP8 model in host RAM on every
  rank at startup.

Measure: `kubectl -n minimax-h3 top pod --containers` during startup and first requests,
then set the limit ~30% above peak.

## CPU

GPU does the heavy work. CPU mainly encodes the finished MP4 (PyAV/libx264; ~1.4 s wall,
~3.2 s CPU time in one benchmark) and preprocesses reference videos with ffmpeg.

- Request 8 cores; 16 is comfortable.
- Prefer **no CPU limit** (throttling stalls encoding unpredictably).

## Disk

| What | Size |
|---|---|
| H3 FL2VA only | ~150 GB volume |
| H3 combined | ~220 GB volume |
| Turbo LoRA adapter | ~2 GB each |
| LTX-2.5 (BF16 file set) | ~70 GB |

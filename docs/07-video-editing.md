# 07 – Video editing

H3 can edit video in two ways, both through `/v1/videos` in vLLM-Omni 0.32.

## 1. Instruction editing (Ref2VA)

Pass one or more input videos as references and describe the change in the prompt.
The official recipe example removes a green-screen background from one video, replaces it
with the environment from a second video, matches the motion, and relights the character.

Good for background swaps, relighting, restyling and compositing.

Requirements:
- The **Ref2VA** partition (`--task-type ref2va`, or combined mode without `--task-type`).
- Reference videos: MP4/MOV, 2–15 s each, ≤ 15 s combined, max 3.
- Serving accepts one image + one audio reference, or one or more videos with no separate
  audio; fewer combinations than the model itself supports.

## 2. Latent-mask editing

Upload the source video plus a mask of what to change:

- `source_video` / `source_audio` uploads
- `video_noise_mask` / `audio_noise_mask` JSON masks: **0 = keep, 1 = regenerate**,
  values in between blend
- 2D mask: applied to every frame (replace an object or region)
- 3D mask `[T, H, W]`: temporal control (continuation/extension)
- Video and audio can be edited independently (e.g. keep picture, regenerate soundtrack).
  Without a separate audio file, the source video's soundtrack is used.

The mask must match the model's latent grid exactly. Follow the "MiniMax H3 Latent-Mask
Editing" section of the vLLM-Omni Videos API docs. Which partition mask editing needs is
not documented: try your FL2VA server first; if rejected, use Ref2VA.

## Limits

- **Short clips:** output 4–15 s; long sources are trimmed, short ones padded with the
  last frame. Edit longer videos in segments.
- **24 FPS** everywhere.
- **Not pixel-exact:** "kept" regions still pass through the VAE, so expect slight changes.
- **Cost:** an edit costs about as much as a new generation, and reference videos make
  sequences longer. A 15 s two-video edit took ~13 min on 4× B300.
- Some features don't combine (e.g. long-video continuation rejects latent-mask editing).

## Deployment implications

| Hardware | Editing setup |
|---|---|
| 2× H100 | Separate Ref2VA server on its own GPU pair |
| 2× H200 | Try combined mode (FL2VA + Ref2VA in one server) |

## Editing with other models

LTX-2.5 edits through task-specific **IC-LoRA** adapters (e.g. deblur, Alpha Gen, the
community "LTX Ripple" first-frame propagation). As of the LTX roadmap issue (updated
2026-10-22), **vLLM-Omni does not support the IC-LoRA pipeline**. Alternatives:
Lightricks' `ltx-pipelines` (`ICLoraPipeline`, distilled model only) behind your own API,
ComfyUI in API mode, or a pending TensorRT-LLM VisualGen pull request.

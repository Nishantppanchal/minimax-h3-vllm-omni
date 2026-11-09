# 10 – Sources

Checked 2026-11-09/10. Where sources disagreed, the newer one was preferred and the conflict
is noted in the relevant doc.

## MiniMax-H3 and vLLM-Omni

- H3 recipe (vllm-omni `main`, most current):
  https://github.com/vllm-project/vllm-omni/blob/main/recipes/MiniMaxAI/MiniMax-H3.md
- Generated recipe page (older, Oct 21): https://recipes.vllm.ai/MiniMaxAI/MiniMax-H3
- vLLM-Omni releases: https://github.com/vllm-project/vllm-omni/releases
- Docker images: https://hub.docker.com/r/vllm/vllm-omni/tags
- H3 roadmap / follow-ups: https://github.com/vllm-project/vllm-omni/issues/5700
- Global FP8 with DLO: https://github.com/vllm-project/vllm-omni/pull/5910
- Latent-mask editing: https://github.com/vllm-project/vllm-omni/pull/7465
- FP8 docs: https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/quantization/fp8/
- Attention backends: https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/diffusion/attention_backends/
- Distributed layerwise offload: https://docs.vllm.ai/projects/vllm-omni/en/latest/design/feature/offloader/distributed_layerwise_offload/
- Videos API: https://docs.vllm.ai/projects/vllm-omni/en/latest/serving/videos_api/
- Intel 2× H200 notes and repo layout: https://huggingface.co/intel-ai/MiniMax-H3-MXFP8-CT-AutoRound-tmp

## Turbo LoRA

- https://huggingface.co/lightx2v/Minimax-h3-Turbo

## LTX-2.5

- vLLM-Omni LTX-2.5 recipe: https://github.com/vllm-project/vllm-omni/blob/main/recipes/LTX/LTX-2.5.md
- LTX support roadmap (IC-LoRA status): https://github.com/vllm-project/vllm-omni/issues/4985
- 2× H200 / 2× H100 benchmarks: https://github.com/mstar-project/mstar/pull/376
- Official facts page: https://ltx.io/llm-info
- Sizes: https://openclawdc.com/blog/can-i-run-ltx-2-5-locally/
- RTX 4090 timings: https://ltx-ai.com/blog/ltx-2-5-review-local-audiovisual-video-generation
- Lightricks pipelines: https://pypi.org/project/ltx-pipelines/

## Kubernetes and tooling

- vLLM Kubernetes deployment: https://docs.vllm.ai/en/stable/deployment/k8s/
- huggingface_hub environment variables: https://huggingface.co/docs/huggingface_hub/main/package_reference/environment_variables

## Model landscape and licenses

- https://fuser.studio/articles/best-open-source-video-models
- https://www.heygen.com/blog/best-video-models

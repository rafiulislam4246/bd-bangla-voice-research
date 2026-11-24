# Olares: TTS/STT Capabilities

Research date: 2026-11-23

## Can Olares fine-tune TTS/STT?

**No built-in capability.** Olares is an INFERENCE platform, not a training platform.

## What Olares has

- **LLMFit**: hardware matcher. Despite "Fit" in name, NOT a fine-tuner - just recommends which pre-trained models fit your RAM/CPU/GPU
- **Model Console** (v1.12.6+): manages inference engines (Ollama, llama.cpp), exposes API endpoints
- **Market AI apps**: all inference-only - Ollama, Open WebUI, AnythingLLM, Gemma4 26B, etc.
- **Agent Skills** (v1.12.6): `olares-chart` wraps Docker Compose/repo into Olares apps

## What Olares DOES NOT have

- No fine-tuning UI
- No TTS/STT training apps
- No Unsloth / Axolotl / LLaMA-Factory packaged
- No GPU scheduling for training jobs
- No training job queue

## Workaround (if insisted)

Via `olares-chart` you can containerize any training framework (F5-TTS, NeMo, Unsloth) as a custom Olares app. But this gives NO benefit over running the same container on bare Linux - no Olares-specific training features.

## Practical workflow

1. **Fine-tune in cloud**: RunPod / Vast.ai / Colab Pro
2. **Store checkpoints**: Olares Drive (good for storage + sync)
3. **Deploy inference in Olares**: wrap mouth.py / ears.py as Olares app, GPU-enabled container
4. **Serve via API**: Olares auto-exposes the ports with domain + SSL

## Olares strengths (what to use it for)

- Persistent local LLM / TTS / STT inference service with public HTTPS URL
- No bills for inference (self-host)
- Dataset storage via Drive
- Multi-app coordination (e.g., ear + brain + mouth as separate apps)

## Sources

- [Olares Market AI category](https://www.olares.com/market/category/AI)
- [LLMFit docs](https://www.olares.com/docs/use-cases/llmfit.html)
- [Olares 1.12.6 release notes](https://www.olares.com/blog/?p=5029)
- [beclab/Olares GitHub](https://github.com/beclab/Olares)

# Fine-Tuning Notes - VERIFIED Only

Research date: 2026-11-23
Scope: F5-TTS (IndicF5_Bangladeshi) and NeMo Conformer-CTC (Bangla ASR)

---

## Does fine-tuning change latency?

**No.** Fine-tune changes weights, NOT architecture.

- F5-TTS: same DiT backbone, same param count (~335M), same checkpoint size, same NFE steps = same inference compute. `F5_CKPT` env var in mouth.py already supports a locally-trained swap.
- Conformer-CTC: same encoder depth, same attention heads, same CTC head = same forward pass cost.

**Verified from upstream**: nothing in F5-TTS inference path or NeMo transcribe path looks at where the checkpoint came from.

---

## Possible SECOND-ORDER latency gain (theoretical, needs verification)

If fine-tune improves model reliability on your specific audio distribution, you MIGHT be able to:
- Lower NFE for F5 while keeping quality - **but repo author already measured NFE=4 and NFE=8 at 8kHz phone channel: 60.4/39.2 and 35.2/16.5 WER/CER** on current base checkpoint. "Speed bought by cutting solver steps is not speed." A fine-tune might change this. Must re-measure with 6 banking lines + 10/10 silence test.
- Use shorter reference clip reliably - depends on fine-tune data diversity

**Do not assume. Measure.**

---

## What fine-tune actually gives

### F5-TTS fine-tune
- Bangladeshi accent accuracy (vs base which leans Indian Bengali)
- Silence failure rate down (current issue: 2/10 vs 10/10 ref clips - base model fragility)
- Domain vocab (banking terms, brand names) pronounced correctly
- Base voice without ref clip becomes usable

### Conformer-CTC fine-tune
- Domain vocab recognition (financial, medical, Bangladeshi slang)
- Accent adaptation if base checkpoint was mostly standard speech
- Phone channel audio robustness (if base was trained on wideband)

---

## Dataset requirements

### F5-TTS
- Official repo: [SWivid/F5-TTS](https://github.com/SWivid/F5-TTS) has finetuning docs
- Minimum for voice-clone fine-tune: not formally documented. Community reports 30min-1hr clean audio for a single speaker fine-tune. Larger datasets give better generalization.
- Format: transcribed audio clips, standard format in `data/` folder

### Conformer-CTC
- NeMo docs: Conformer-CTC fine-tuning tutorial exists
- Conservative guidance: 10+ hours transcribed audio for meaningful accent adaptation. Less than that risks overfitting.
- Format: NeMo manifest JSON (audio path + duration + text)

### Repo author's VITS warning (applies broadly)
- VITS fine-tune on 12.3 min = 28.3% CER vs base 1.9% (catastrophic)
- Small dataset on unfamiliar architecture = worse than base, every time
- F5-TTS is more robust to small data than VITS (flow matching + voice clone design), but still: **more data is always better**

---

## Where to fine-tune

### NOT Olares
- Olares is INFERENCE platform only (verified in olares-tts-stt.md)
- No fine-tuning UI, no training-job scheduler, no pre-packaged training frameworks
- Possible via custom containerized app (`olares-chart`), but no benefit over bare Linux

### Cloud options (ranked by F5-TTS compatibility)
- **RunPod** ($0.50-2/hr H100) - supports CUDA + PyTorch, fastest to set up for F5-TTS
- **Vast.ai** - cheaper, less reliable
- **Modal** - serverless, pay-per-second, good for small runs
- **Colab Pro** ($10/mo) - runtime limits (24hr max), ok for short fine-tunes
- **Local** - needs RTX 4090+ with 24GB+ VRAM for F5-TTS training

### Rule
**Fine-tune in cloud, deploy inference in Olares.** The `F5_CKPT` env var is there for exactly this.

---

## Training stacks (verified)

### F5-TTS
- Official: https://github.com/SWivid/F5-TTS - has training script in `src/f5_tts/train/`
- Finetune doc: in repo README `## Training` section

### Conformer-CTC
- Official: https://github.com/NVIDIA/NeMo
- Tutorials: NeMo ASR tutorials notebook has a Conformer-CTC finetune walkthrough
- Command: `python speech_to_text_finetune.py --config-path=... init_from_nemo_model=...`

### Framework helpers that do NOT help here
- **Unsloth** - LLM only
- **Axolotl** - LLM only
- **LLaMA-Factory** - LLM only
- These three are commonly recommended for "fine-tuning" but have ZERO TTS/STT support

---

## Cost estimate (ballpark, not production quote)

### F5-TTS fine-tune
- GPU: 1x H100 on RunPod, ~$2/hr
- Dataset prep + training: 8-16hrs on good data
- Total: **$15-35**
- Output: single checkpoint ~1.3GB, swap via `F5_CKPT` env in mouth.py

### Conformer-CTC fine-tune
- GPU: 1x A100 on RunPod, ~$1.50/hr
- Dataset prep + training: 20-40hrs
- Total: **$30-80**
- Output: NeMo checkpoint, swap in ears.py `init_from_nemo_model` path

---

## Checklist before starting

1. Dataset: at least 30 min clean Bangladeshi audio with accurate transcripts for F5-TTS, 10 hrs+ for Conformer
2. Hold-out test set separated BEFORE training (never trained on)
3. Baseline numbers recorded: current CER/WER + silence failure rate on the hold-out
4. Regression gate defined: fine-tuned model must beat baseline on both axes, not just one
5. Deploy plan: how to swap checkpoint in production safely (A/B route first, then full switch)

---

## Sources

- [F5-TTS repo](https://github.com/SWivid/F5-TTS)
- [NeMo ASR docs](https://docs.nvidia.com/nemo-framework/user-guide/25.09/nemotoolkit/asr/models.html)
- Olares inference-only capability confirmed in [olares-tts-stt.md](olares-tts-stt.md)
- Repo author measurements (VITS fine-tune catastrophic) in `bdvoice-stt-tts-master/FINAL.md`

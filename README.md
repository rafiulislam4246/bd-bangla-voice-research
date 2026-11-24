# TTS/STT Research - Bangladeshi Bangla Voice Stack

Research notes collected on 2026-11-23 about Bangla STT/TTS, specifically for Bangladeshi Bangla (bn-BD).

Source project: `bdvoice-stt-tts-master` (Downloads folder).

## Files

- [current-stack.md](current-stack.md) - Jeta akhon use hocche (ears + mouth) - from repo author's own measurements
- [bd-bangla-tts-alternatives.md](bd-bangla-tts-alternatives.md) - IndicF5 er compare, ki ache ki nai - HF search
- [bd-bangla-asr-alternatives.md](bd-bangla-asr-alternatives.md) - bn-regional-ct2 er compare (Whisper medium bn fine-tune) - commercial license filtered
- [latency-optimization.md](latency-optimization.md) - **VERIFIED only** - F5-TTS repo + NeMo docs te confirm kora items, RISKY/UNKNOWN items alada warning section e
- [fine-tuning.md](fine-tuning.md) - **VERIFIED only** - F5-TTS + Conformer fine-tune, latency impact, cost
- [olares-tts-stt.md](olares-tts-stt.md) - Olares fine-tune / run capability
- [verbex-voice-agent-architecture.md](verbex-voice-agent-architecture.md) - Verbex full stack: brain/ear/mouth coordination, state machine, tool calling, flow agent nodes. Reverse-engineered Oct-Nov 2026.

## Verification policy

**Every optimization claim verified against upstream source code or official docs.** Items that could not be verified are marked UNKNOWN or RISKY and must NOT be applied without local testing against the 6 banking lines CER/WER + 10/10 silence test.

Research methodology:
1. Read repo author's own measurements in `bdvoice-stt-tts-master/FINAL.md` and `mouth.py` comments
2. Verify each optimization claim against F5-TTS repo (SWivid/F5-TTS `utils_infer.py`) and NeMo docs/PRs
3. Only mark CONFIRMED items as safe wins. Everything else flagged.

## Quick verdict (Nov 2026)

- **Best STT**: `kazalbrur/bangla-asr-conformer-120m-dialects` (43-105ms, 4.9% CER, Apache-2.0)
- **Best TTS**: `actorcritic/IndicF5_Bangladeshi` (350-426ms NFE=8, MIT, zero-shot voice clone)
- **IndicF5 theke better bn-BD TTS akhono nai** open-source e - space te investment kom
- **TTS verified wins**: ref-tensor-in-memory + trim ref to 6-10s + streaming for long text. torch.compile / flash-attn NOT confirmed for F5-TTS - don't enable blind.
- **STT verified wins**: bf16 inference (NVIDIA blog), client-side 16k resample, VAD edge-trim. Mid-utterance chunking + flash-attn + INT8 RISKY for offline Conformer.
- **Fine-tune latency**: same (architecture unchanged). Quality may improve, enabling lower NFE if re-measured.

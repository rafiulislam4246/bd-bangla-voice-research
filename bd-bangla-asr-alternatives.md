# Bangladeshi Bangla ASR Alternatives (Nov 2026)

Research date: 2026-11-24
Baseline: `bengaliAI/tugstugi_bengaliai-regional-asr_whisper-medium` via faster-whisper (CTranslate2)
- CER: ~4.4%, WER: ~19.6%, Latency: ~346ms, Apache-2.0, streams NO

Target: commercial license (NOT CC-BY-NC, NOT research-only), better accuracy OR streaming.

## Verdict

**Nothing open-source + commercially-licensed clearly beats your current 4.4% CER / Whisper-medium setup on all axes.** Only ONE candidate worth bench time: **ai4bharat/indic-conformer-600m-multilingual** as a streaming tier.

## Worth testing (1)

### ai4bharat/indic-conformer-600m-multilingual
- **URL**: https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual
- **Architecture**: Conformer CTC + RNNT hybrid (NeMo)
- **License**: **MIT** (commercial OK)
- **Bangla WER/CER**: NOT published on model card (only Hindi 13.2 on Vaani listed)
- **Streaming**: YES - NeMo Conformer-RNNT supports cache-aware streaming. **Architecturally the only real streaming candidate in open-licensed set.**
- **Pan-Indic**, not BD-specific - BD accent performance UNKNOWN
- **Pros**: potential streaming tier, MIT license, newer architecture, RNNT head
- **Cons**: unverified on BD, pan-Indic bias (may have West Bengal + other Indic lean)

**Test plan**: bench on your BD test clips with NeMo cache-aware streaming. If Bangla CER lands within ~1 point of 4.4% AND streams under 200ms chunks → use as streaming tier alongside current ear.

## Skip (checked, not better)

### BuzzASR/bengali
- URL: https://huggingface.co/BuzzASR/bengali
- Whisper-large-v3 full fine-tune, Sept 2026, **MIT**
- CER 5.5% / WER 19.42% on CommonVoice-25 (WORSE than your 4.4%)
- CER 10.16% / WER 31.97% on FLEURS
- Pan-Bengali (no BD-accent focus)
- ~2B params → slower than your medium
- No streaming
- **Not a win on any axis**

### OmiKhan/bangla-asr-whisper-medium
- URL: https://huggingface.co/OmiKhan/bangla-asr-whisper-medium
- Whisper-medium fine-tune, ~300h public data
- **License NOT declared** on card - unusable commercially without confirmation
- Same architecture + size as yours
- Not demonstrably better

### NVIDIA Canary-1B-v2 / Parakeet-TDT-0.6B-v3
- Released Sept 2026, CC-BY-4.0 / NVIDIA OM
- Lists 25 languages but **Bengali NOT confirmed** in per-language WER table
- NeMo page indicates Hindi/Tamil/etc Indic focus
- No BD evaluation

### NVIDIA Nemotron ASR Streaming
- Streaming conformer-transducer
- **English-only** - rules out

### Moonshine
- **English-only**

### Seamless M4T v2
- Includes Bengali
- **CC-BY-NC-4.0 - BLOCKED for commercial use**

### Sarvam saarika:v2.5, Soniox v5, Deepgram nova-3
- Lead the riverborn BanSpeech benchmark (18.2-23.9% WER)
- **All closed/hosted, not open weights**
- Deepgram US endpoint penalty already analyzed in `pipeline-streaming-architecture.md`

### Hishab titu_stt / FastConformer / 18k-hour
- Verbex's own training stack
- **All CC-BY-NC-4.0 - BLOCKED**

### BUET / DU / Bengali.AI 2026
- Nothing new surfaced beyond the tugstugi regional line already in use
- No new Bengali.AI model card since the regional Whisper drop

## Recommendation (locked)

**Keep `tugstugi_bengaliai-regional-asr_whisper-medium` as the accuracy anchor.**

Only one experiment worth engineering time:
- Bench `ai4bharat/indic-conformer-600m-multilingual` on BD test clips with NeMo cache-aware streaming
- If Bangla CER within ~1 point of 4.4% AND streams under 200ms chunks → use as **streaming tier** alongside current ear
- Current ear stays as high-accuracy final-pass; new one handles interim-transcript streaming for Pattern 1 of pipeline streaming

Everything else is a regression or a license trap.

## Bench protocol if testing indic-conformer-600m-multilingual

1. Install NeMo: `pip install nemo_toolkit[asr]`
2. Download model: `nemo_asr.models.EncDecHybridRNNTCTCBPEModel.from_pretrained('ai4bharat/indic-conformer-600m-multilingual')`
3. Run on existing 200-clip BD test set → record CER/WER
4. Enable cache-aware streaming config → measure interim-chunk latency
5. Regression gate:
   - CER within 1 point of 4.4% baseline ✓
   - Streaming chunk latency < 200ms ✓
   - 10/10 silence test PASS (adapt from TTS test methodology)
6. Only then add as parallel ear

## Note on Verbex

Verbex uses Deepgram in production, NOT their own Hishab models (which exist but are CC-BY-NC locked). This confirms that even Hishab themselves route around their research models for commercial scale - cost/latency/multilingual coverage of Deepgram wins at their size. For us (smaller scale, BD-only, cost-sensitive), self-hosted Whisper medium is the right trade.

## Sources

- [BuzzASR/bengali](https://huggingface.co/BuzzASR/bengali)
- [ai4bharat/indic-conformer-600m-multilingual](https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual)
- [AI4Bharat ASR page](https://ai4bharat.iitm.ac.in/areas/asr/)
- [OmiKhan/bangla-asr-whisper-medium](https://huggingface.co/OmiKhan/bangla-asr-whisper-medium)
- [riverborn bangla-stt-benchmark-results](https://huggingface.co/datasets/riverborn/bangla-stt-benchmark-results)
- [Canary-1B-v2 & Parakeet-TDT-0.6B-v3 paper](https://arxiv.org/html/2509.14128v2)
- [Open ASR Leaderboard 2026](https://arxiv.org/html/2510.06961v1)
- [Whisper vs Wav2Vec-BERT on Bangla](https://arxiv.org/pdf/2507.01931)

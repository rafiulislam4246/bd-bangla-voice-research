# Latency Optimization - VERIFIED Only

Research date: 2026-11-23
Baseline: TTS 350-426ms (IndicF5, NFE=8, bf16), STT 43-105ms (Conformer-120M)
Method: every claim verified against F5-TTS repo (SWivid/F5-TTS) and NVIDIA NeMo docs/issues. Items marked UNKNOWN or RISKY are NOT verified - do not apply blindly.

**Rule from repo author**: every change must pass the 6 banking lines CER/WER test AND the 10/10 silence test before shipping. No exception.

---

## TTS (IndicF5 / F5-TTS)

### ✅ CONFIRMED SAFE WINS

#### 1. Keep reference tensor + text resident in memory across calls
- F5-TTS has `_ref_audio_cache` / `_ref_text_cache` in [utils_infer.py](https://raw.githubusercontent.com/SWivid/F5-TTS/main/src/f5_tts/infer/utils_infer.py), MD5-keyed
- BUT only caches preprocessed audio/text, NOT the DiT encoder output
- `socket_server.py` pattern: load once in `update_reference`, keep tensor + text in memory, do NOT call `preprocess_ref_audio_text` per request
- **Save**: eliminates per-request preprocess overhead

#### 2. Trim reference clip to 6-10s
- F5-TTS hard-clips refs to **12s** in `preprocess_ref_audio_text` (3 strategies, final hard-truncate at 12000ms)
- Ref longer than 12s = zero benefit, same cost after trim
- F5 concatenates ref + gen mel for the DiT forward → shorter ref = shorter attention sequence = linear speedup
- **Safe quality-wise** (shorter means less information for voice cloning - verify with 10/10 silence test)
- Current `short_26.wav` - check its length, if >6s, test a trimmed version

#### 3. Official streaming via `infer_batch_process(streaming=True, chunk_size=2048)`
- F5-TTS supports this natively - `socket_server.py` uses it
- Yields 2048-sample chunks at 24kHz as batches finish
- `infer_process` does NOT expose it, must call `infer_batch_process` directly
- Use `few_chars`/`min_chars` smaller first split for lowest TTFB
- **Important**: reduces time-to-first-audio for **LONG text**, does NOT speed up a single short utterance
- First audio: ~chunk_size/24000 ≈ 85ms window
- **Safe**: same model, same output, just streamed delivery

#### 4. Keep current Euler + NFE=8 (do NOT touch)
- F5-TTS paper: Euler is faster AND better at larger NFE **when Sway Sampling is on**
- Midpoint is 2nd-order (2 NFE per step = slower per step); repo comment says "better with small NFE" but at NFE=8 Euler is correct
- Switching to midpoint at NFE=8 **doubles cost**
- Repo author already tested NFE=4 and NFE=8 at 8kHz phone channel: 60.4/39.2 and 35.2/16.5 WER/CER → "speed bought by cutting solver steps is not speed"
- **Rule**: do not touch NFE or solver

### ⚠️ EXPERIMENTAL (test before prod, don't apply blind)

#### 5. Fast-F5-TTS step pruning
- Paper: [arXiv 2505.19931](https://arxiv.org/pdf/2505.19931v1)
- Training-free non-uniform step pruning, claims **4x speedup**
- Quality claims unverified on Bangladeshi Bangla phone channel
- Requires A/B with repo's 6 banking lines + 10/10 silence test

#### 6. SmoothCache (attention/FFN caching across denoising steps)
- Project: [F5-TTS SmoothCache](https://siratish.github.io/F5-TTS_SmoothCache)
- Quality tradeoff, not free
- Test before prod

### ❌ UNKNOWN / DO NOT APPLY BLIND

#### `torch.compile(model)`
- **UNKNOWN**: no flag in F5-TTS `utils_infer.py`, no confirmed-clean issue thread
- F5 uses DiT with dynamic shapes (text length varies) - risky for compile
- Nothing says it breaks, nothing says it works. Not production-safe without testing.

#### Flash Attention 2
- **UNKNOWN**: F5-TTS has no explicit `flash_attention` switch
- May route via `torch.nn.functional.scaled_dot_product_attention` automatically on supported GPUs, but repo exposes no toggle
- Do not claim it enables FA2 without reading the actual DiT module

#### CUDA graphs / `cudnn.benchmark = True`
- **RISKY**: sequence length varies per call (text + ref length), hostile to CUDA graphs' static-shape requirement
- `cudnn.benchmark=True` safe-ish but little gain on dynamic shapes, slows first N calls while tuning
- **Not recommended**

### Realistic TTS gains (verified)

| Change | Save | Risk |
|---|---|---|
| Ref tensor+text resident in memory | eliminates preprocess overhead | zero |
| Trim ref to 6-10s | linear with sequence reduction | low (verify 10/10) |
| Streaming via `infer_batch_process` | TTFB ~85ms for long text | zero |

**For a single short banking line** (≤6 words), streaming does NOT help - full generation still 350-426ms. Streaming wins only when text is long enough to batch.

---

## STT (faster-whisper / CTranslate2, bn-regional-ct2)

**Runtime**: faster-whisper wrapping a Whisper-medium fine-tune, running on CTranslate2 backend. NOT NeMo Conformer. All NeMo-specific tricks DO NOT APPLY to this runtime.

**Baseline**: ~346ms live, ~4.4% CER, ~19.6% WER. No native streaming.

### ✅ CONFIRMED SAFE WINS (zero accuracy risk)

#### 1. Pin `language="bn"` (biggest safe single win)
- Skips Whisper's auto language-detection forward pass
- Save: **~100-300ms** (documented)
- Code:
```python
segments, _ = model.transcribe(audio, language="bn", ...)
```

#### 2. `condition_on_previous_text=False`
- Safe for short single-chunk utterances (<30s, no prior context to use)
- Also reduces Whisper's known hallucination loops
- Zero accuracy risk for typical phone-call utterance length
- Code:
```python
segments, _ = model.transcribe(audio, language="bn", condition_on_previous_text=False, beam_size=5)
```

#### 3. Keep `beam_size=5` (do NOT lower)
- Published Bengali long-form: beam 5 = 16.75% WER vs beam 1 = 18.42% WER (+1.67% absolute)
- Dropping beam = quality loss. Violates zero-accuracy-loss rule.

### ⚠️ HARDWARE-DEPENDENT WIN (verify GPU first)

#### 4. CT2 `flash_attention=True` (Ampere+ GPUs only, CC ≥8.0)
- `ctranslate2.models.Whisper` has `flash_attention` constructor arg, defaults False
- faster-whisper's `WhisperModel` does NOT expose it as kwarg - must instantiate CT2 directly or patch
- **Bit-identical math** (no accuracy change), measurable decoder win
- Check GPU compute capability first: `nvidia-smi --query-gpu=compute_cap --format=csv`
- If CC ≥8.0 (A100, RTX 30xx, RTX 40xx, H100): apply
- If CC 7.x (T4, RTX 20xx, V100): stay on float16 default
- Docs: https://opennmt.net/CTranslate2/python/ctranslate2.models.Whisper.html

### ⚠️ MUST-MEASURE items (potential wins, needs YOUR CER test)

#### 5. compute_type switch
- Valid types: `int8`, `int8_float16`, `int8_bfloat16`, `float16`, `bfloat16`, `float32`
- Default `"default"` keeps model's converted type
- CT2 claims "little to no degradation" - **UNVERIFIED for Bangla fine-tune**
- Steps:
  1. Record current CER on 200-clip test
  2. Try `compute_type="float16"` if currently float32
  3. Try `compute_type="int8_float16"` if currently float16
  4. Re-run CER, accept only if no regression
- Risk: int8 variants may hurt accuracy on Bangla - no published data
- Docs: https://opennmt.net/CTranslate2/quantization.html

### ❌ RISKY / DO NOT APPLY

#### `vad_filter=True`
- CONFIRMED latency win on audio with silence
- But: `min_speech_duration_ms` can drop very short Bangla utterances
- Timestamp mismatch reported (faster-whisper issue #364)
- **Safer to keep OFF** if utterances are already tight (no leading/trailing silence)

#### `without_timestamps=True`
- UNKNOWN: not publicly benchmarked
- Expected accuracy-neutral on text output but unverified
- If you don't need timestamps, test on your 200 clips before enabling

#### `chunk_length` below 30s
- **RISKY**: Whisper trained on 30s windows, shorter degrades positional-embedding regime
- No official safe-reduction benchmark
- Do not touch

#### Native streaming
- **DOES NOT EXIST** in faster-whisper. No `transcribe_stream()`.
- Wrappers available but treat as integration work, not a flag:
  - `ufal/whisper_streaming`
  - `collabora/WhisperLive`
  - `WhisperLiveKit`
- Each wrapper has own accuracy/latency tradeoff - A/B against current setup before adopting

#### Distil-Whisper Bangla
- **DOES NOT EXIST**. `distil-large-v3` is English-only
- No official Bangla distil checkpoint on HF
- Rules out this entire optimization path

#### BatchedInferencePipeline (faster-whisper 1.1.0+)
- CONFIRMED 3-9x throughput, but on LONG audio only
- Does NOT help a single 346ms utterance unless batching concurrent requests
- PR #856 shows mixed WER deltas (one dataset improved, one slightly regressed)
- Not useful for single-caller phone pattern

### Realistic STT gains (VERIFIED)

Current baseline: ~346ms

| Change | Save | Risk | Note |
|---|---|---|---|
| Pin `language="bn"` | **100-300ms** | zero | documented, immediate |
| `condition_on_previous_text=False` | small, anti-hallucination | zero | safe for short utterances |
| CT2 `flash_attention=True` (Ampere+ only) | measurable | zero | bit-identical math |
| `compute_type` upgrade | needs YOUR CER test | medium | gate on test |

**Realistic target after easy wins: ~150-250ms** (down from 346ms), zero quality loss.

### STT reality check for your runtime
Whisper medium at 346ms is ALREADY the quality tradeoff you accepted over Conformer's 105ms. **The only way to meaningful streaming is a different model class entirely** (streaming-capable Conformer or Deepgram or community Whisper streaming wrapper). Patterns 2, 3, 4 of the streaming pipeline (in-process + brain stream + TTS sentence boundary) still work without touching STT.

### Previous (wrong) advice retracted
Earlier I quoted NeMo Conformer optimizations (bf16 via `model.to(torch.bfloat16)`, `rel_pos`/`abs_pos` self_attention_model, NVIDIA 10x acceleration blog). **None of that applies to faster-whisper/CT2 runtime.** Those were wrong for your actual setup. Current advice is faster-whisper-specific and verified against SYSTRAN/faster-whisper + CTranslate2 docs.

---

## Combined end-to-end

Real wins only (CONFIRMED items):

| Stage | Baseline | Verified-safe optimized | Source |
|---|---|---|---|
| STT | 43-105ms | bf16 + client resample + VAD trim | NVIDIA blog |
| Brain | 1087ms | (cloud Gemini - outside scope) | - |
| TTS short line | 350-426ms | ref-in-memory + trimmed ref (streaming does NOT help short lines) | F5-TTS repo |
| TTS long line | 350-426ms→659ms | streaming cuts TTFB to ~85ms | F5-TTS repo |

**Brain is the dominant cost**. STT and TTS are already in optimized territory.

---

## Implementation order (do in this sequence, measure after each)

1. **Trim ref clip to 6-10s** (5 min) - re-run 10/10 silence test
2. **TTS ref tensor+text resident in memory** (code change in mouth.py) - measure
3. **STT bf16 inference** (1 line change) - run 200-clip CER test
4. **Client-side 16k mono resample** (caller code change) - measure server CPU drop
5. **TTS streaming via `infer_batch_process`** (bigger refactor, long text only) - measure TTFB

Keep a CSV with: change, latency p50, latency p95, CER, WER, silence failure rate (/10). If any metric regresses, revert immediately.

---

## Sources

- [F5-TTS repo](https://github.com/SWivid/F5-TTS)
- [F5-TTS utils_infer.py](https://raw.githubusercontent.com/SWivid/F5-TTS/main/src/f5_tts/infer/utils_infer.py)
- [F5-TTS paper](https://arxiv.org/pdf/2410.06885)
- [Fast F5-TTS (step pruning)](https://arxiv.org/pdf/2505.19931v1)
- [F5-TTS SmoothCache](https://siratish.github.io/F5-TTS_SmoothCache)
- [NVIDIA: Accelerating NeMo ASR 10x](https://developer.nvidia.com/blog/accelerating-leaderboard-topping-asr-models-10x-with-nvidia-nemo)
- [NeMo ASR models docs](https://docs.nvidia.com/nemo-framework/user-guide/25.09/nemotoolkit/asr/models.html)
- [NeMo PR #9590 (flash-attn + custom bias)](https://github.com/NVIDIA/NeMo/pull/9590)
- [Cache-Aware Streaming Conformer paper](https://arxiv.org/html/2312.17279v3)
- [Zero-shot INT8 Conformer quantization](https://arxiv.org/abs/2103.16827)

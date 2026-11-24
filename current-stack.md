# Current Stack (bdvoice-stt-tts-master, 22 Nov 2026)

Source: `~/Downloads/bdvoice-stt-tts-master.zip`
Replacing: a service billed at 8 taka per minute.

## STT (Ear) - ACTUAL current config

| Field | Value |
|---|---|
| Default EARS | **`bn-regional-ct2`** (ears.py line 153) |
| Base model | `bengaliAI/tugstugi_bengaliai-regional-asr_whisper-medium` - Whisper medium fine-tuned on Bangladeshi regional dialects |
| Runtime | faster-whisper (CTranslate2 backend), via `IamSanjid/tugstugi_bengaliai-regional-asr_whisper-medium-ct2` |
| Licence | Apache-2.0 |
| Benchmark CER | ~4.4% (within 0.2 of bn-regional vanilla 4.2%) |
| Benchmark WER | ~19.6% |
| Benchmark latency | 346 ms live, 876 ms for non-CT2 variant |
| Streams | **NO** (Whisper is batch-only, blocks Pattern 1) |
| Port | 8300 |
| Start | `python ears.py` (defaults to bn-regional-ct2) |

### Full ear benchmark table (from ears.py lines 20-23)

| Model | WER | CER | Latency | Streams |
|---|---|---|---|---|
| bn-regional (vanilla Whisper) | 19.6% | 4.2% | 876ms | NO |
| **bn-regional-ct2** (current default) | ~19.6% | ~4.4% | ~346ms | NO |
| bn-whisper | 23.1% | 5.6% | 575ms | NO |
| conformer-120m | 26.1% | 5.6% | 51ms | **YES** |
| fastconformer-116m | 26.1% | 9.7% | 50ms | YES |

### Live mic test (ears.py lines 141-142)
- **bn-regional-ct2 (346ms)**: "ালামু আলাইকুম। আমার কথা কি শুনো যায়?" - loses first syllable only
- conformer-120m (105ms): "আহাললাহ সাল্লাহু আলাইহওয়াসম্মার কথা কি শুনা যায়" - invents sentence

### Why bn-regional-ct2 is the default
Repo author chose accuracy over latency: WER 19.6% vs 26.1%, CER 4.4% vs 5.6%, real-world transcript loses 1 syllable vs invents whole sentences.

The trade-off cost: **3x slower (346 vs 105ms), cannot stream.**

## TTS (Mouth)

| Field | Value |
|---|---|
| Model | `actorcritic/IndicF5_Bangladeshi` |
| Licence | MIT |
| Settings | NFE=8, bf16, speed=1.15, ref=`short_26.wav` |
| Latency | 350-426 ms short/medium, 659 ms long |
| Accuracy | 9.7% WER, 6.9% CER on 6 banking lines |
| Voice change | 4-sec reference clip, no training needed |
| Port | 8301 |
| Start | `MOUTH=indicf5 F5_NFE=8 F5_SPEED=1.15 python mouth.py` |

**VITS rejected**: 44-74ms (8x faster) BUT single fixed voice. Fine-tune on 12.3 min = 28.3% CER vs base 1.9%. Needs 10-25 hours per speaker to fine-tune properly.

## End-to-end turn cost (with current ACTUAL ear)

| Stage | ms |
|---|---|
| Ear (bn-regional-ct2) | ~346 |
| Brain (cloud Gemini) | 1,087 |
| Mouth (IndicF5) | 350-426 |
| **Total serial** | **~1,783-1,859** |

For comparison (real phone calls): Telnyx 1296ms, ElevenLabs 1424, Bland 1520, Vapi 1558, Retell 1740. None under 1 second. Verbex own: 800-1450ms. Our serial total already above competitors.

## Critical gotchas from repo author

1. **Env var is `EARS` not `EAR`** - wrong name silently ignored, default loads, health route lies
2. **Three separate venvs required** - Coqui TTS and `f5_tts` pin incompatible versions
3. **Reference clip selection**: NEVER pick by accuracy alone. 10x repeat, count silence failures FIRST, accuracy after. `kks_today.wav` won on accuracy, failed 8/10 with silence. `short_26.wav` = 10/10 reliable.
4. **F5_REF_TEXT** wrong transcript = worse voice, NO error
5. **`F5_CKPT` env var** available for locally fine-tuned F5 checkpoint swap

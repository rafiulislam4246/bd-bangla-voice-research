# Pipeline Streaming Architecture - How to Get Verbex-Class Latency

Date: 2026-11-24
Context: Our voice agent runs ~800-1600ms end-to-end. Verbex-class is 800-1200ms with better feel. The gap is not STT or TTS - it's **serial pipeline vs streaming pipeline**.

---

## The pattern

```
STT final ──► Brain starts
              Brain first token ──► TTS starts speaking first word
                                    While Brain still generating
                                    While TTS speaking earlier words
```

This is NOT one optimization. It's **four distinct patterns stacked**:

1. STT streams interim transcripts
2. Brain starts on interim (or starts immediately on final, no HTTP overhead)
3. Brain streams tokens one-by-one
4. TTS starts synthesizing on first sentence boundary, not full response

Each pattern needs its own setup. You can adopt them one at a time.

---

## What this saves (math)

### Current (serial)

```
User speaks ─────────────────────────────────────┐
                                                  │
                User stops                        │
                     │                            │
                     ▼                            │
            STT runs (105ms)                      │
                     │                            │
                     ▼                            │
            Brain runs (1087ms) ◄── user waiting silently
                     │
                     ▼
            TTS runs (350ms)
                     │
                     ▼
            First audio reaches user
            ────────────────────────► ~1600ms wait
```

### Streaming (pipeline)

```
User speaks ─────────────────────────────────────┐
                                                  │
     Interim transcript "আমার পেমেন্ট..."        │
                     │                            │
                     ▼ (speculative)              │
            Brain starts generating ◄── saves ~500ms
                     │
                User stops
                     │
            STT final arrives
                     ▼
            Brain validates context, continues
                     │
                     ▼
            Brain streams first sentence
                     │
                     ▼
            TTS starts on first sentence
                     │
                     ▼
            First audio ────────────► ~400-600ms perceived wait
            ─────────────────────────────────────┐
            While Brain finishes rest            │
            While TTS speaks first sentence      │
            Brain completes                       │
            TTS speaks remaining sentences        │
```

**Perceived latency drop**: 1600ms → 400-600ms for first audio. Full response still takes same real time, but user hears sound within ~500ms.

---

## Can we use this pattern in our voice agent?

**Answer: PARTIALLY YES, with specific limits our stack imposes.**

Per-pattern breakdown:

### 1. STT streaming with interim transcripts
**Our status: NOT POSSIBLE without model change.**

Why: Our current STT is `kazalbrur/bangla-asr-conformer-120m-dialects` which is an **offline NeMo Conformer-CTC** (verified in `latency-optimization.md`). Offline Conformer-CTC does NOT support true streaming interim results. Buffered chunking gives accuracy loss at boundaries.

To get this pattern, we need to SWITCH STT model. Options:
- **Deepgram** (vendor, 3rd party API) - [see verification below once research back]
- **NeMo Cache-Aware Streaming FastConformer** - different model class, trained with limited right-context, Bangla version availability UNKNOWN
- **Whisper streaming** (various community streaming wrappers) - accuracy OK on Bangla, latency variable

**Impact if adopted**: ~300-500ms perceived wait reduction.

### 2. Brain starts immediately on STT final
**Our status: POSSIBLE NOW, no model change.**

How: eliminate HTTP serialization between ears.py → brain. Run them in the same Python process using LiveKit Agents SDK pattern.

Current architecture (ears.py port 8300, brain somewhere, mouth.py port 8301) requires:
- HTTP POST from ears → brain = serialize JSON + network RTT + parse = 20-50ms
- HTTP POST from brain → mouth = another 20-50ms
- Total HTTP overhead: 40-100ms

In-process pipeline: direct Python function call, 1-2ms.

**Impact**: ~40-100ms real latency reduction.

### 3. Brain streams tokens
**Our status: POSSIBLE NOW, depends on brain model.**

Cloud Gemini: supports streaming via `generate_content_stream` or Google GenAI streaming API.
OpenAI: `stream=True` native.
Any local LLM via vLLM / llama.cpp: native streaming.

Just need to switch from `response = brain.call(prompt)` (blocking) to `for token in brain.stream(prompt): ...` (streaming).

**Impact alone**: zero, because text tokens are invisible. But it UNLOCKS pattern #4.

### 4. TTS starts on first sentence boundary
**Our status: POSSIBLE NOW, needs code change in mouth.py.**

Logic:
```
accumulate tokens from brain stream
on sentence-ending punctuation (। ? ! ...):
   dispatch accumulated sentence to TTS
   continue accumulating next sentence
```

F5-TTS can handle this via `infer_batch_process(streaming=True, chunk_size=2048)` (verified from upstream). Each sentence becomes one batch, audio chunks flow as each batch completes.

**Impact**: for multi-sentence responses, first audio arrives after FIRST sentence generates + synthesizes, not full response. For 3-sentence Bangla response of ~15 words total:
- Brain first sentence: ~300-400ms
- TTS first sentence: ~350ms
- First audio to user: **~650-750ms**
- vs serial waiting for full response: ~1500ms+

**For single-sentence short responses, this helps little** (same as baseline, as noted in earlier research).

---

## Realistic adoption path

### Phase 0 (do NOW, zero model change)
1. **Combine ears.py + brain + mouth.py into one LiveKit Agents Python process**
   - Eliminates HTTP serialization
   - Direct function handoff
   - Save 40-100ms
2. **Switch brain call to streaming API**
   - Single flag change if cloud Gemini / OpenAI
   - Zero perceived impact until #3
3. **Change mouth.py to accumulate tokens + dispatch per sentence**
   - Use F5-TTS `infer_batch_process(streaming=True)`
   - Save 300-500ms perceived for multi-sentence responses

**Phase 0 total save for multi-sentence responses**: **400-700ms perceived**
**Phase 0 total save for short responses**: **40-100ms real**

### Phase 1 (STT switch, requires validation)
4. **Switch STT to a streaming-capable Bangla model** (Deepgram being researched - see verification note below)
5. **Start brain on interim transcript** (speculative) - accept ~5-10% waste when user corrects mid-utterance

**Phase 1 additional save**: 300-500ms perceived

### Phase 2 (realtime model, biggest single win)
6. **Replace separate STT + Brain + TTS with a realtime model** like `gpt-realtime` or `gemini-2.0-flash-live-001`
   - Audio in, audio out
   - No STT/TTS calls at all
   - Verbex supports this as a Brain provider option

**Phase 2 save**: brain latency 1087ms → 200-400ms = **~700ms real**
**But**: loses our Bangladeshi-accent TTS (realtime models have their own voices, Bangla accent varies)
**Trade-off**: speed vs accent quality. Not zero-cost.

---

## Our stack vs Verbex, per pattern

| Pattern | Us (current) | Us (possible) | Verbex |
|---|---|---|---|
| STT streaming interim | ❌ offline Conformer | ⚠️ needs model switch | ✅ Deepgram |
| In-process pipeline | ❌ HTTP between services | ✅ do now | ✅ LiveKit Agents |
| Brain token streaming | ❌ blocking call | ✅ do now | ✅ |
| TTS sentence boundary | ❌ full response | ✅ do now | ✅ |
| Realtime model | ❌ | ⚠️ loses our TTS accent | ✅ offered |

**Three of five patterns are available to us with zero model risk.**

---

## Deepgram STT for our pipeline - VERIFIED Nov 2026

### Language support
- **Model**: Nova-3 only (monolingual + multilingual variants). Nova-2, Enhanced, Base do NOT support Bangla.
- **Code**: `bn` only - **no `bn-BD` vs `bn-IN` distinction**. One model for all ~230M Bangla speakers.
- **GA/Beta status**: UNKNOWN - not stated in Deepgram docs
- **Release**: "earlier in 2026", accuracy update 7 Aug 2026
- Docs: https://developers.deepgram.com/docs/models-languages-overview

### Latency (CRITICAL finding)
- Marketing claims `<300ms` real-time but NOT Bangla-specific
- Bangla time-to-first-interim: **UNKNOWN** (no published numbers)
- **NO Asia endpoint** - Deepgram's documented regions are US-based (`api.deepgram.com`)
- **Bangladesh → US RTT: ~250-350ms added to inference**
- **Our current Conformer: 43-105ms self-hosted from BD = unbeatable on latency**

### Streaming
- WebSocket `/v1/listen?model=nova-3&language=bn` supported
- `interim_results=true` standard flag - works for Bangla per docs
- Bangla-specific streaming limitations: UNDOCUMENTED

### Accuracy on Bangla - the dealbreaker
- **Deepgram-published Bangla WER: NONE found** (Asia-Pacific benchmarks only cover Thai/Chinese)
- Independent benchmarks vs Whisper / IndicASR / Conformer on Bangla: NONE found
- No code-mixing (bn-en) performance data
- No phone channel (8kHz, G.711) tuning evidence
- No Bangladeshi-accent-vs-West-Bengal breakdown
- **Comparing to our 4.9% CER on 200 clips is IMPOSSIBLE without running Deepgram on the same clips**

### Pricing
- Nova-3 monolingual: $0.0048/min PAYG, $0.0042/min Growth promo
- Nova-3 multilingual: $0.0058/min PAYG, $0.0050/min Growth promo
- `bn` likely falls under monolingual tier (not explicitly stated)

### BD-specific risks
- No evidence Deepgram trained on Bangladeshi-accent data distinct from West Bengal
- No evidence of phone-channel (8kHz) Bangla tuning
- One shared `bn` model for both countries - **assume West-Bengal bias until proven otherwise**

### Verdict: DO NOT SWITCH on current public data

Reasons:
1. **Latency worse**: 43-105ms self-hosted vs ~250-350ms RTT alone to US, before inference
2. **Accuracy unknown**: no published Bangla WER means we trade a measured 4.9% CER for an unknown
3. **Accent risk**: West Bengal bias likely, no BD-specific tier
4. **Cost adds up**: ~$0.005/min = $0.30/hour of audio. Meaningful at scale.
5. **Interim streaming benefit lost to RTT**: 300ms saved by streaming interim is eaten by 300ms US RTT = net zero

### How to properly evaluate IF curious
1. Sign up Deepgram free tier
2. Run `nova-3 / language=bn` on our exact 200-clip test set
3. Measure CER vs our baseline 4.9%
4. Measure BD→US RTT from our deployment location
5. If CER better AND total latency (RTT + inference) better than 43-105ms, THEN consider migration
6. Otherwise keep Conformer

**Streaming pattern from this doc still works without Deepgram** - we can stream Brain + TTS without touching STT. Patterns 2/3/4 unlocked without any STT change.

### Sources
- [Deepgram Models & Languages](https://developers.deepgram.com/docs/models-languages-overview)
- [Nova-3 Asia-Pacific expansion blog](https://deepgram.com/learn/nova-3-adds-12-new-languages-across-europe-and-south-asia)
- [Deepgram Bengali product page](https://deepgram.com/product/speech-to-text/bengali)
- [Deepgram pricing](https://deepgram.com/pricing)
- [Improved Models changelog 2026-08-07](https://developers.deepgram.com/changelog/2026/8/7)

---

## Implementation checklist for Phase 0 (ready to do)

File-by-file changes needed in `bdvoice-stt-tts-master`:

### 1. Merge ears.py + brain + mouth.py into one process
- Use LiveKit Agents Python SDK
- Install: `pip install livekit-agents livekit-plugins-openai`
- Create `agent.py` that wires: NeMoSTT → LLM → F5TTS
- Single port replaces three ports (8300, 8301, 8302)

### 2. Brain streaming
- If cloud Gemini: `model.generate_content_stream(prompt)` instead of `model.generate_content(prompt)`
- If OpenAI: `client.chat.completions.create(..., stream=True)`

### 3. Sentence-boundary TTS dispatch
Pattern:
```python
buffer = ""
for token in brain.stream(prompt):
    buffer += token
    if any(buffer.endswith(p) for p in ["।", "?", "!", "...", "।\n"]):
        sentence = buffer.strip()
        buffer = ""
        await tts_queue.put(sentence)  # mouth picks it up, synthesizes
```

In mouth.py: replace `infer_process` with `infer_batch_process(streaming=True, chunk_size=2048)`.

### 4. Measurement before/after
Record for 10 representative responses:
- `time_to_first_audio_ms` (TTFA) - caller-perceived
- `time_to_last_audio_ms` (TTLA) - real completion
- Silence failure rate (10/10 test)
- CER on same 6 banking lines

**Regression gate**: if CER regresses, revert. If silence failure rate exceeds 1/10, revert.

---

## Open questions (do not commit until answered)

1. Deepgram Bangla WER/CER on our 200-clip test set - **research in flight**
2. Does realtime model (gpt-realtime / gemini-live) accent sound acceptable in Bangla vs current F5+ref clip? - needs A/B
3. LiveKit Agents SDK + F5-TTS integration - any official plugin? Or build custom? - **unverified**
4. Olares deployment: can a LiveKit Agents Python worker run as an Olares app? - probably yes via custom container, needs trial

---

## Sources

- F5-TTS `infer_batch_process(streaming=True)` - verified in `latency-optimization.md`
- Conformer-CTC offline-only confirmed - verified in `latency-optimization.md`
- LiveKit Agents framework - https://docs.livekit.io/agents/
- Verbex per-call flow - verified in `verbex-voice-agent-architecture.md`

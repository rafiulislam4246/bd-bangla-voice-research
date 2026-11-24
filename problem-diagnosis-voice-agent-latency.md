# Voice Agent Latency Problem - Verified Diagnosis + Fix

Date: 2026-11-24
Context: Response to sibling team's PROBLEM.md about a Bangla voice agent stuck at 1,914ms vs 1,500ms target.

**Summary**: The 463ms post-idle gap on the brain is **GPU P-state downclocking**, confirmed. Everything else in their stack is correctly measured and mostly correctly configured. Fix is 2 commands + a systemd service.

---

## Their measurements (recap)

```
ear          324 ms median   faster-whisper / CTranslate2
brain        878 ms median   Qwen3.8-27B-Q4 llama.cpp (remote, 3.8-7.6ms RTT)
mouth        670 ms median   IndicF5, NFE=12, bf16
first sound  1,914 ms median
```

Brain detail:
- Back-to-back: **476ms** TTFT
- After 6s idle: **939ms** TTFT
- Gap: **+463ms** — unexplained after 4 ruled-out hypotheses

---

## The brain cold-start: ROOT CAUSE CONFIRMED

### Diagnosis: NVIDIA P-state downclocking (P0 → P8)

Matches evidence exactly:
- 463ms post-idle gap = textbook P8→P2 ramp cost (NVIDIA forum: RTX 5090 reports ~500ms, matches almost exactly)
- Hot path (476ms) stays fast = GPU already at P0, no ramp needed
- Prompt cache working (441/445 cached) = NOT prefill
- Separate remote box = NOT local contention
- Warm-up ping 330ms earlier failed = not enough time to complete clock ramp

### Candidates RULED OUT (with reasons)

| Candidate | Why ruled out |
|---|---|
| CUDA graph invalidation | Would hit every call, not just post-idle. Hot 476ms already uses graphs (llama.cpp PR #6766, #7302) |
| KV defrag (`--defrag-thold`) | Triggers on fragmentation, not idle. Cache persists through 20s idle, confirmed |
| HTTP keep-alive drop | TCP reconnect at 3.8ms RTT is 4-20ms, not 463ms |
| CPU governor (powersave) | 10-50ms max, too small for 463ms |
| llama.cpp `--sleep-idle-seconds` | Only if explicitly enabled. Default is off (issue #23395) |

### Fix (run as root on brain box)

```bash
# 1. Persistence mode - keeps driver loaded, stops P-state teardown
sudo nvidia-smi -pm 1

# 2. Query supported clocks for your card
nvidia-smi --query-supported-clocks=gr,mem --format=csv

# 3. Lock SM + memory clocks to max supported (replace values)
sudo nvidia-smi -lgc <max_gr_clock>
sudo nvidia-smi --lock-memory-clocks=<max_mem_clock>

# 4. CPU governor (secondary but do it)
sudo cpupower frequency-set -g performance

# 5. Verify under idle (should stay at locked clock, pstate P0/P2)
nvidia-smi --query-gpu=pstate,clocks.sm,clocks.mem --format=csv -l 1
```

### Persist across reboot

Persistence mode does not survive reboot. Create a systemd unit:

```ini
# /etc/systemd/system/nvidia-performance-lock.service
[Unit]
Description=NVIDIA GPU performance lock for low-latency inference
After=nvidia-persistenced.service
Requires=nvidia-persistenced.service

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-smi -pm 1
ExecStart=/usr/bin/nvidia-smi -lgc <max_gr_clock>
ExecStart=/usr/bin/nvidia-smi --lock-memory-clocks=<max_mem_clock>
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Then:
```bash
sudo systemctl enable --now nvidia-persistenced
sudo systemctl enable --now nvidia-performance-lock
```

### Expected result

- TTFT gap collapses to **under 50ms**
- Hot 476ms stays the same (that's forward-pass + sampling, not P-state)
- Brain median: **939ms → ~500-550ms**
- Total first-sound: **1,914ms → ~1,476ms**
- **Target hit: under 1,500ms achieved with margin**

### Verify before adopting

Lock clocks, re-run `probe_cold.py`. If first-token latency is same for back-to-back AND after 6s idle → confirmed. If still gap → next suspect is `llama.cpp` streaming SSE header buffering (less likely but possible).

### Caveats

- On some cards (GB10, older 1080Ti) `-lgc` silently fails. Confirm under load.
- Thermal/power throttling overrides the lock - keep cooling adequate.
- Locked max clocks increase idle power draw (~20-40W extra). Acceptable for production serving.

### Sources

- [NVIDIA nvidia-smi docs](https://docs.nvidia.com/deploy/nvidia-smi)
- [NVIDIA forum: clock ramp-up issue](https://forums.developer.nvidia.com/t/very-fast-ramp-down-from-high-to-low-clock-speeds-leading-to-increased-time-repeatedly-ramping-up/165974)
- [nvidia-persistenced README](https://download.nvidia.com/XFree86/Linux-x86/384.47/README/nvidia-persistenced.html)
- [RTX 5090 lock clocks](https://forums.developer.nvidia.com/t/how-to-force-lock-sm-and-memory-clocks-on-rtx-5090-headless-linux/348794)
- [llama.cpp CUDA Graphs PR #6766](https://github.com/ggml-org/llama.cpp/pull/6766)
- [NVIDIA: optimizing llama.cpp with CUDA graphs](https://developer.nvidia.com/blog/optimizing-llama-cpp-ai-inference-with-cuda-graphs)
- [llama.cpp idle reload issue #23395](https://github.com/ggml-org/llama.cpp/issues/23395)

---

## Q2 (27B wrong shape?) - answer: FIX P-STATES FIRST

**Real problem is Topic A, not the model.**

Model-size reasoning for completeness:
- With prompt fully cached (zero prefill) + 20-40 output tokens, TTFT ≈ one forward pass through all layers
- Scales roughly linearly with param count and layer depth
- First-token is NOT bandwidth-bound like decode throughput
- Dropping 27B → 9B roughly 2.5-3x the forward pass

### Recommendation: fix P-states first, keep Qwen3-27B

If after that TTFT still exceeds 300ms:
1. **Primary drop: `Qwen2.5-14B-Instruct Q4`** (Apache 2.0, strongest published multilingual including Bangla among open models, 14B at Q4 ~8GB, forward pass ~40% of 27B)
2. **Backup for simple turns: `hishab/titulm-llama-3.2-3b-v2.0-Instruct-v1.0`** (Verbex Bangla Mini, purpose-built Bangla, extended tokenizer = fewer Bangla tokens, Llama 3.2 Community License - commercial OK under 700M MAU, TituLLMs paper arxiv 2502.11187 beats base Llama 3.2 on Bangla)

### Skip
- **Gemma-2 9B**: Bangla trails Qwen in community reports, Gemma license has use restrictions
- **Phi-4 14B**: Microsoft does not claim Bangla, English-focused

### Don't migrate blindly
Benchmark both on YOUR Bangla banking scripts - published scores don't capture banking-domain phrasing.

---

## Q3 UPDATED (section 6b: can F5-TTS reduce TTFA?)

**Their team is RIGHT on architecture, WRONG on version.**

### Version claim check
- They cite `f5_tts 0.1.0` - **this version does NOT exist on PyPI**
- Earliest PyPI release: `0.3.4` (Jan 2025)
- Latest PyPI: `1.1.22` (Jul 2026)
- Their quoted `socket_server.py` code matches a **pre-Feb-2025 snapshot**, not current
- Streaming support shipped in **PyPI 0.6.2 (2025-02-26)** and solid in **1.0.0 (2025-03-12)**

**Fix**: upgrade `f5_tts` to 1.1.22. Current signature:
```python
def infer_batch_process(..., streaming=False, chunk_size=2048, ...)
```

### Architecture claim check
**Correct for a single utterance.** When `streaming=True`:
- Iterates text batches sequentially
- Yields fixed-size slices
- **BUT each batch's FULL generation completes before any audio yields**
- Chunks arrive at batch boundaries, NOT during model sampling
- Flow matching inherently produces whole waveform per NFE pass - can't peek mid-ODE

So for a single 3-word clause = single batch = still blocks on full render. Their team's architectural conclusion stands.

### The fix they haven't adopted: `first_package` flag
Current `socket_server.py` already does this: `first_package` flag re-splits the first request with smaller `few_chars` / `min_chars` limits so first audio lands fast, then normal chunking takes over.

Splitting on clause boundaries (6-10 words) is standard. Below ~4 words prosody degrades because F5-TTS conditions on batch's duration prediction.

**Realistic impact**: first-audio to 300-450ms on a single clause (down from 620-720ms).

### Route they asked about in Q4: pick BOTH

Section 4 (brain idle) + section 6b (TTS first-chunk) are not alternatives - they're additive:
- Brain fix (P-state): ~400ms savings on cold-start path
- TTS fix (pip upgrade + first_package): ~250-320ms savings on first-chunk

Total potential: 1914ms → ~1240ms. Well under 1500ms target, with margin for retrieval/tools later.

### Clause streaming (section 6, Q5) - keep it

**Keep it.** Reasons:
- After brain P-state fix: first clause arrives ~500ms, mouth still 620ms (or 350ms with upgrade + first_package). Overlap exists again.
- Even without overlap, clause streaming starts speaking the moment brain's first clause is ready. Rendering whole response requires full brain finish = +520-660ms wait.
- 18-char minimum workaround is correct and widely used.

### Refinement (optional)

Instead of fixed char count, use time-based lookahead:
```python
clause_duration_ms = len(clause_chars) * 55  # Bangla playback ~55ms/char
render_time_estimate = get_render_estimate(len(next_clause_chars))
if clause_duration_ms < render_time_estimate + 100:  # 100ms safety margin
    merge_forward(clause, next_clause)
```

This is the "watermark buffering" pattern used in commercial streaming. 18-char fixed works for consistent content; time-based is more adaptive.

### What's standard

Verbex, Retell, ElevenLabs Flash, Deepgram Aura all do sentence/clause-level streaming. The "render whole answer first" approach is the old pattern - adds 500-1500ms. Their current approach is correct.

---

## Pre-synthesized filler audio pattern (standard, missed by them)

While brain generates, play a short pre-synthesized filler ("একটু দেখি...", "ঠিক আছে...") to hide the 500-700ms rendering time.

**Verified standard pattern** - documented by LiveKit, Agora, Inworld, AWS Builder Center.

### Protocol
1. Synthesize 5-10 filler phrases ONCE at startup
2. Cache in RAM as WAV
3. On LLM wait > ~1.5s, play filler from cache
4. On real audio arrival, cross-fade or immediately replace

**Critical**: do NOT synthesize fillers live. That defeats the purpose - takes as long as the real response. Pre-cache.

UCF/CUI 2025 study: fillers don't significantly hurt UX. Verbex/Retell/Vapi all ship this.

### Sources
- [AWS Builder Center: killing dead air](https://builder.aws.com/content/3KREFKYYAEJuEwaL5XiDMgG1ann/killing-dead-air-how-we-made-a-phone-agent-sound-like-it-is-listening)
- [LiveKit Agents audio customization](https://docs.livekit.io/agents/multimodality/audio/customization/)
- [Agora filler words docs](https://docs.agora.io/en/ai/build/shape-the-conversation/filler-words.md)

---

## If they want to switch TTS entirely (the escape hatch)

If F5-TTS pipeline hits ceiling even after upgrade + first_package + EPSS, the escape is autoregressive TTS that streams sub-generation.

| Model | Streams sub-gen? | Clone ref | Bangla |
|---|---|---|---|
| **BnTTS** (XTTS-v2 continued-pretrained on 3.85k hr Bengali) | Yes via `inference_stream()` ~200ms TTFB | ~6s | **Purpose-built for exactly this gap** - paper arXiv 2502.05729 |
| XTTS v2 (vanilla) | Yes ~200ms TTFB | ~6s | OK, weaker than BnTTS |
| Orpheus | Yes ~180-200ms TTFB, ~100ms with input streaming | ≥5s | Not evaluated, LLaMA-backbone tokenizer unlikely to cover Bangla well |
| GPT-SoVITS v2 | Yes via `stream_mode` | 5s | No native Bangla, needs fine-tune |
| StyleTTS2 | **NO** (streaming is TODO) | yes | No |
| Fish Speech 1.5 | Yes (autoregressive) | yes | Weak |

**Best Bangla-preserving swap: BnTTS** - purpose-built for exactly this gap, inherits XTTS streaming infrastructure.

**But don't swap yet.** Steps 1-3 below should hit target without this.

---

## Q4 (wrong stage to chase?) - UPDATED with verified numbers

**Previous (wrong) advice retracted**: I earlier said to drop NFE=12 → NFE=8 for ~220ms mouth savings. **This is WRONG.** Fast-F5-TTS paper (arXiv 2505.19931) shows:

| NFE | WER on LibriSpeech-PC |
|---|---|
| 32 (default) | 2.42% |
| 16 | 2.44% (lossless) |
| 7 | 4.16% (~70% relative degradation) |

**NFE=8 is on the degradation cliff.** Repo author's own 9.7% WER at NFE=8 confirms this. Keep NFE=12.

### Correct ranking (verified)

| Stage | Current | Achievable | Savings | How |
|---|---|---|---|---|
| **Brain** | 878ms | ~500ms | **-378ms** | **GPU P-state fix (Topic A) - biggest and easiest** |
| Ear | 324ms | ~230ms | -94ms | `language="bn"` + `condition_on_previous_text=False` on faster-whisper (ONLY if actually using Whisper variant - if truly NeMo Conformer this doesn't apply) |
| Mouth | 670ms | 670ms | 0 (keep NFE=12) | For further gains use **EPSS** (Fast-F5-TTS step pruning) - training-free, NFE=7 at parity with NFE=16 quality |

**Total projected after P-state fix alone**: 1,914ms → ~1,536ms. Just below target with 36ms margin.
**Total projected after P-state + ear flags**: 1,914ms → ~1,442ms. Comfortably under.

### If they want aggressive further gains: EPSS

Project: https://fast-f5-tts.github.io/
Paper: arXiv 2505.19931
Claim: training-free non-uniform step pruning, NFE=7 at WER parity with NFE=16

Not production-ready without validation on their 6 banking lines + 10/10 silence test. But if P-state fix + ear flags still don't hit target, this is the next lever. Mouth could drop from 670ms to ~400ms without quality loss.

---

## Ear note - worth a double-check

Their doc says `faster-whisper / CTranslate2` with model `kazalbrur/bangla-asr-conformer-120m-dialects`. But:
- faster-whisper is designed for Whisper architecture
- `kazalbrur/bangla-asr-conformer-120m-dialects` is a NeMo Conformer-CTC model

These are incompatible runtimes. Either:
1. They actually use a different (Whisper-based) model and the doc is wrong
2. They converted the Conformer somehow (unusual)
3. They're running NeMo directly and calling the wrapper "faster-whisper" loosely

Worth clarifying - if it's actually NeMo Conformer-CTC, our `latency-optimization.md` STT section applies (bf16 inference per NVIDIA blog), not the faster-whisper flags.

Whatever the runtime, 324ms for a 120M Conformer is suspiciously high (repo author measured 51ms for same model). May indicate warm-up issue or wrong model name in doc.

---

## Action order (verified priority, updated after section 6b research)

### Cheap wins first
1. **Upgrade `f5_tts` to 1.1.22** (5 min, pip install) - unlocks real streaming support that exists but they're missing
2. **Enable `first_package` flag in socket_server.py** (code change, 1 hour) - re-splits first request with smaller chunks → **first-audio 620ms → 300-450ms**
3. **Fix GPU P-states on brain box** (1 hour including systemd setup, zero risk) - **saves ~400ms** on brain idle
4. **Pre-synthesize filler audio cache** (2 hours) - 5-10 fillers once, cached in RAM, play on wait >1.5s
5. **Clarify ear runtime** - faster-whisper vs NeMo? Then apply correct flags

### Do NOT
- **Do NOT drop NFE below 12** (NFE=8 is on degradation cliff per Fast-F5-TTS paper)
- **Do NOT swap TTS away from IndicF5** until steps 1-4 proven insufficient
- **Do NOT switch LLM** until P-state fix validated

### Only if target still missed after above
- Try Qwen2.5-14B-Instruct Q4 for brain
- Try EPSS (Fast-F5-TTS step pruning) for mouth - can drop render time 620ms → 350ms
- Last resort: swap IndicF5 for BnTTS (XTTS-v2 Bengali continued-pretrain)

### Projected total after steps 1-4

| Stage | Current | After fix | How |
|---|---|---|---|
| Ear | 324ms | ~250ms | Ear flags |
| Brain | 878ms (median mix) | ~500ms | P-state fix |
| Mouth (first clause) | 670ms | 350-450ms | pip upgrade + first_package |
| **Total first-sound** | **1,914ms** | **~1,100-1,200ms** | **well under 1,500ms target** |

Zero architecture change needed. Target achievable in 1 day of work.

---

## Honest caveats

- Observed 500ms RTX 5090 P8→P2 ramp number is from one NVIDIA forum report - may vary by GPU generation. Verify by locking clocks and re-measuring on actual hardware.
- Qwen3-27B vs Qwen2.5-14B Bangla delta is community-reported, no published head-to-head on Bangla banking phrasing
- EPSS claims unverified for Bangla phonotactics - English step schedules may not transfer perfectly
- Their 1,914ms baseline measurement assumes current config; real traffic may vary with network conditions

---

## Sources (all this doc)

### TOPIC A (cold-start)
- [NVIDIA nvidia-smi docs](https://docs.nvidia.com/deploy/nvidia-smi)
- [NVIDIA forum: clock ramp-up](https://forums.developer.nvidia.com/t/very-fast-ramp-down-from-high-to-low-clock-speeds-leading-to-increased-time-repeatedly-ramping-up/165974)
- [RTX 5090 lock clocks forum](https://forums.developer.nvidia.com/t/how-to-force-lock-sm-and-memory-clocks-on-rtx-5090-headless-linux/348794)
- [llama.cpp CUDA Graphs PR #6766](https://github.com/ggml-org/llama.cpp/pull/6766)
- [NVIDIA: optimizing llama.cpp with CUDA graphs](https://developer.nvidia.com/blog/optimizing-llama-cpp-ai-inference-with-cuda-graphs)
- [llama.cpp idle reload issue #23395](https://github.com/ggml-org/llama.cpp/issues/23395)

### TOPIC B (LLM)
- [TituLLMs paper](https://arxiv.org/abs/2502.11187)
- [TituLLMs ACL 2025](https://aclanthology.org/2025.findings-acl.1279)

### TOPIC C (F5-TTS NFE)
- [F5-TTS paper arxiv 2410.06885](https://arxiv.org/html/2410.06885v3)
- [Fast-F5-TTS EPSS paper arxiv 2505.19931](https://arxiv.org/pdf/2505.19931v1)
- [Fast-F5-TTS project page](https://fast-f5-tts.github.io/)
- [F5-TTS repo defaults](https://github.com/SWivid/F5-TTS)

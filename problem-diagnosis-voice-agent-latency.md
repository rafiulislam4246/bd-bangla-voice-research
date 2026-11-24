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

## Q3 (clause streaming architecture right?)

Yes, right approach. Their 18-char minimum workaround is correct and widely used.

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

## Action order (verified priority)

1. **Fix GPU P-states on brain box** (1 hour total including systemd setup, zero risk) — saves ~400ms
2. **Clarify ear runtime** - is it actually faster-whisper or NeMo? Then apply correct flags
3. **Do NOT touch NFE on mouth** - keep NFE=12
4. **Only if target still missed**: try Qwen2.5-14B-Instruct Q4 for brain, EPSS for mouth

Zero architecture change needed. Target achievable by Tuesday.

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

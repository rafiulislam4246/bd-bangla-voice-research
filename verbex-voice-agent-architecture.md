# Verbex Voice Agent - Brain / Ear / Mouth Architecture

Source: reverse-engineered Oct-Nov 2026 via CDP capture + Mintlify docs + GitHub forks + 422-driven API probing.
Full details: `/Users/rafiulislam/Work/n8n/bangla-voice/VERBEX-ARCHITECTURE.md` (613 lines)
Flow API surface: see memory [[verbex-flow-agent-api-surface]]

---

## The 3 pieces and how they fit

```
       ┌─────────────────────────────────────────────┐
       │   Caller (phone or browser)                 │
       └──────────────────┬──────────────────────────┘
                          │ WebRTC audio/opus+RED
                          ▼
    ┌──────────────────────────────────────────────┐
    │  LiveKit Cloud (SFU)                         │
    │  pia-platform-dev-k9sck0q7 (PROD confirmed)  │
    │  Region: Chicago ochicago1a                   │
    │  SIP: 2sv4xh8d6zi.sip.livekit.cloud          │
    └──────────────────┬───────────────────────────┘
                       │ dispatches worker per room
                       ▼
    ┌──────────────────────────────────────────────┐
    │  Agent Worker (Python, LiveKit Agents SDK)   │
    │  Identity: agent-<ID>, name="v2"             │
    │  State: listening → thinking → speaking      │
    │  Track: roomio_audio                          │
    └──────────────────────────────────────────────┘
       │            │           │           │
       ▼            ▼           ▼           ▼
    EAR         BRAIN         MOUTH     TOOLS
```

---

## EAR - Deepgram (streaming STT)

**Not their own model in production.** Hishab has their own `titu_stt_bn_fastconformer` released on HF, but prod uses Deepgram.

- **Provider**: Deepgram WebSocket streaming
- **Model**: Nova family (nova-2-general, nova-2-phonecall, nova-2-conversationalai, etc.)
- **Languages**: bn, en, ja
- **Pattern**: audio frames streamed in realtime, interim + final transcripts stream back
- **Why not own model**: cost/latency/multilingual coverage - Deepgram wins on all three vs self-hosting Conformer

**Takeaway**: Even Verbex (10 yr Bangla NLP house) chose a third-party STT. Their own Conformer serves research purposes only.

---

## BRAIN - LLM layer (how it understands questions)

### Provider enum (4 first-class)
`openai | anthropic | verbex | openai_compatible`

**`verbex` is a first-class provider** - their own `Verbex Bangla Mini` (hishab/titulm-llama-3.2-3b-v2.0-Instruct-v1.0) runs native, NOT bolted on.

### Models available
**Standard (text LLMs):**
- GPT-4.1 (1M context, highest intelligence)
- GPT-4.1-mini (default for most agents, balanced)
- GPT-4.1-nano (fastest, simple tasks)
- GPT-4o (default in Agentic Agents)
- Groq Llama 3.3 70B / 3.1 8B (ultra-fast tool-use)

**Realtime (voice-native, bypass STT+TTS):**
- OpenAI `gpt-realtime`
- Google `gemini-2.0-flash-live-001` / `gemini-live-2.5-flash-preview`

Realtime models skip Deepgram and Pia TTS entirely - audio in, audio out from the LLM.

### How "right answer" happens

Three agent types with different orchestration:

| Agent type | Mechanism |
|---|---|
| **Simple Agent** | One prompt + one LLM. Linear chat. For FAQ bots. |
| **Agentic Agent** | Tool-calling. LLM decides when to call APIs (Gmail, calendar, transfer, custom). Default GPT-4o. |
| **Flow Agent** | Visual flow-builder with 11 node types: `start, conversation, collection, variable-extraction, math-operation, call-api, decision, transfer, sms, press-digit, end-call`. Per-node LLM/STT/TTS override. Deterministic paths. |

### Context injection
- `pia_llm_dynamic_data` field on call creation - arbitrary JSON injected into LLM context at call start (user data, cart, order history, campaign variables)
- Pre-call warm context eliminates "who is this caller" LLM queries

### Knowledge Base (RAG for domain answers)
- **Vector DB**: Pinecone (default). Enum also allows qdrant, weaviate, milvus, pgvector
- **Embedding**: `text-embedding-3-large` (OpenAI), **immutable per KB** (changing invalidates vectors)
- **Chunking**: `recursive` default, 512 chars / 50 overlap
- **Retrieval**: hybrid (semantic + keyword) for better recall
- **Upload formats**: PDF, DOCX, TXT, MD, XLSX, XLS, website crawl
- **Async ingestion**: 202 Accepted → worker chunks+embeds+indexes → poll until `completed`
- **No update endpoint**: every submission mints a new `document_id`. To refresh: delete + resubmit (not atomic)

---

## MOUTH - Pia/Verbex in-house TTS (the real moat)

### What it is
- Base architecture: **Coqui XTTS v2** (CONFIRMED from `github.com/verbex-ai/BnTTS/README.md` quote: *"Building upon the XTTS architecture, our approach integrates Bangla into a multilingual TTS pipeline"*)
- 3.85k hours Bangla pretrain (3,636 hr pseudo-labeled + rest from LIMMITS/OpenSLR/CRBLP/in-house)
- Few-shot speaker adaptation
- Paper: [arXiv 2502.05729 (BnTTS)](https://arxiv.org/abs/2502.05729)
- **Weights NOT released** - this is the moat

### API surface (from `github.com/verbex-ai/verbex-speech-sdk/SPEC.md`)
- `POST /v1/speak/*` - batch TTS
- `WS /v1/speak/stream` - multi-utterance streaming TTS
- `GET /v1/speak/voices` - returns all 18 voices with sample URLs
- `WS /v1/listen/stream` - live STT (16kHz mono s16le PCM)

### 18 named voices
Maria, Rashida, Hena, Niloy, Rafiq, + 13 others. All Bangladeshi accent, male and female, all Bangla + English (English added July 2026).

### Known limits (inferred from their prompt-level instructions)
Verbex Bangla Mini model card shows agents are instructed:
- Say "পনেরো" instead of "১৫" (number normalization)
- Preserve emails verbatim
- For URLs say "ওয়েবসাইট" prefix

**Signal**: Their TTS does NOT have strong SSML / text-normalization built in. Commercial TTS (Azure, Google, ElevenLabs) handle this at engine level. Verbex handles it at prompt level.

### `pronunciations` field
`[{id, term, replacement}]` at agent level. Highest-value feature for Bangla - Bengali product names are in no TTS training set. On `simple` agent creation it accepts; on `flow` creation it comes back empty (bug or undocumented).

---

## State machine - how brain/ear/mouth coordinate per utterance

Observed from live CDP capture, LiveKit protobuf frames:

```
listening ──► (user speaks, Deepgram final transcript arrives)
    │
    ▼
thinking  ──► (transcript → LLM; LLM may call tools)
    │          │
    │          ▼ (if tool called)
    │       [TTS speaks tool_start_message while tool runs]
    │          │
    │          ▼ (tool result injected back into LLM context)
    │
    ▼
speaking  ──► (final LLM response text → Pia TTS → audio chunks published)
    │
    ▼
listening ──► (next turn)
```

State published as `lk.agent.state = listening | thinking | speaking` metadata events so frontend shows visual feedback.

### Per-utterance latency budget

| Stage | Latency |
|---|---|
| Mic frame → Deepgram first chunk | 100-200 ms |
| Deepgram final → LLM first token | 200-500 ms |
| LLM first token → TTS first chunk | 200-400 ms |
| TTS first chunk → user ear | 50-100 ms |
| **End-to-end** | **550-1200 ms** |

Chicago region adds ~250ms RTT for BD users → total **800-1450 ms**.

---

## Tool calling (how it ACTS on understood intent)

**OpenAI function-calling schema** (via `/v2/ai-agents/{agent_id}/custom-tools/`):

```json
{
  "tool_id": "string",
  "openai_tool": {
    "name": "transfer_to_support",
    "description": "...",
    "parameters": { "type": "object", "properties": {...} }
  },
  "additional_tool_info": {
    "url": "https://...",
    "method": "POST",
    "headers": {...}
  },
  "tool_start_message": "Let me check that for you...",
  "tool_delayed_message": "Still looking...",
  "tool_complete_message": "Found it.",
  "tool_failed_message": "Sorry, couldn't look that up."
}
```

TTS speaks the lifecycle messages while HTTP call runs - **no silent waiting**. This is a UX pattern worth copying.

### Built-in tools
- Send Email (Gmail)
- Call Transfer (cold + warm, with whisper message)
- Calendar (Google appointment booking)

### Custom integrations
- Direct HTTP endpoints
- Make.com webhook
- Zapier webhook

---

## How a flow agent "understands and responds correctly"

Flow Agent is their deterministic path - the brain is constrained by the graph, not free-form LLM.

**11 node types:**
`start, conversation, collection, variable-extraction, math-operation, call-api, decision, transfer, sms, press-digit, end-call`

**Key pattern:**
- `conversation` nodes: free LLM chat within scope (mode = `full | static-only | prompt-only`)
- `collection` nodes: extract specific slots (name, number, date) with validation
- `variable-extraction` nodes: parse structured data from utterance
- `decision` nodes: branch on variable values
- `transfer` nodes: cold / warm-non-agentic / warm-agentic transfer to human

**7 edge kinds:**
`session-start, direct-edge, conditional-edge, on-reply-edge, logic-branch, logic-default, error-fallback`

Every edge needs `contextPassing` (what variables flow through). Every collection step needs `voiceSettings: {source: "flow-default"}` placeholder.

### Vendor validates almost nothing
The Verbex API accepts graphs with unreachable nodes, paths that run off the end, decisions branching on variables nothing fills. Repo author (Kotha platform) caught one in production (`কাশফুল ফ্লো (উন্নত)`) that would have transferred every caller to a human. **Layer 2 validation must be yours** - Kotha does this in `bangla-voice/platform/convex/flowgraph.ts`.

---

## What's defensible and what's wiring

| Asset | Public? | Replaceable? |
|---|---|---|
| LiveKit Cloud | Yes (anyone) | Trivial |
| LiveKit Agents framework | Open source | Trivial |
| Deepgram | Public API | Trivial |
| GPT-4.x | Public API | Trivial |
| Verbex Bangla Mini LLM | Published on HF | **You can download and run** |
| **Pia TTS (18 voices)** | NOT public | **The moat** |
| Hishab's 10 yr Bangla data | NOT public | Impossible to replicate fast |

**Engineering moat = TTS + training data.** Everything else is a wiring job.

---

## Agent-level config fields often missed

From 422-driven API probing (see [[verbex-flow-agent-api-surface]]):

- `pronunciations` - highest value for Bangla
- `dynamic_data_config`
- `end_call_instruction`
- `max_endpointing_delay` - how long to wait for user to keep speaking
- `vad_activation_threshold`
- `wait_time_before_detecting_end_of_speech`
- `ambient_sound_volume` - background noise to make caller comfortable
- `amd_voicemail_message`
- `voicemail_detection_timeout_seconds`
- `graph_data.agentDefaults.llm.llm_id` - **required for flow create**, missing it = 422

The last one is the trap - nothing in a graph read back from them marks it as mandatory.

---

## What to copy if building your own voice agent

### Must-copy patterns
1. **State machine with metadata events** (listening/thinking/speaking) - gives frontend visual hook
2. **Tool lifecycle messages** (start/delayed/complete/failed) - TTS speaks while HTTP runs, no silence
3. **Dynamic data injection at call start** - warm context, no mid-call lookups
4. **Pronunciations table at agent level** - Bangla product names require this
5. **Hybrid retrieval** (vector + keyword) - better recall than pure semantic
6. **Per-agent voice settings with per-node override** (flow agents) - different sections can have different voices/speeds
7. **Realtime models as a provider** - skip STT+TTS entirely when latency matters most

### Avoid their mistakes
1. **Validate graph correctness in your own layer** - vendor accepts broken graphs
2. **Pick a region close to your users** - Chicago is wrong for BD; Singapore/Tokyo better
3. **Use SSML-capable TTS where possible** - avoid their prompt-level normalization workaround
4. **Separate dev and prod LiveKit projects** - their `-dev` suffix on prod is confusing

---

## For our use case (bdvoice-stt-tts-master)

Current stack uses:
- **Ear**: Conformer-120M (self-hosted) - Verbex uses Deepgram instead for a reason (cost/latency/multilingual)
- **Brain**: cloud Gemini (per FINAL.md) - Verbex offers Gemini realtime live
- **Mouth**: IndicF5_Bangladeshi (self-hosted) - Verbex uses their own XTTS v2 fine-tune

**Our edge**: fully self-hosted, zero vendor bills, Bangladeshi accent via IndicF5
**Verbex's edge**: streaming + low latency across the stack, 18 pre-made BD voices, visual flow builder, mature tool-calling

**Phased path to match Verbex quality** (from `bangla-voice/VERBEX-ARCHITECTURE.md` Section 7):
1. **Phase 1 (this week)**: ship with Deepgram + GPT-4.1-mini + Azure bn-BD. Costs $0.03-0.08 per 3-min call.
2. **Phase 2 (3-4 wk)**: fine-tune `kenpath/svara-tts-v1` (Orpheus Indic, Apache-2.0, zero-shot) OR `ai4bharat/IndicF5` on 10-20 hrs BD recordings.
3. **Phase 3 (optional)**: self-host `hishab/titulm-llama-3.2-3b-v2.0-Instruct-v1.0` for LLM.

---

## Sources

- Full architecture: `/Users/rafiulislam/Work/n8n/bangla-voice/VERBEX-ARCHITECTURE.md`
- Flow API surface memory: [[verbex-flow-agent-api-surface]]
- BnTTS paper: [arXiv 2502.05729](https://arxiv.org/abs/2502.05729)
- Verbex public repos: https://github.com/verbex-ai (44 public, including BnTTS, titu-stt, titulm, coqui forks)
- Verbex speech SDK: https://github.com/verbex-ai/verbex-speech-sdk
- Verbex JS SDK: https://github.com/verbex-ai/verbex-js-sdk (hardcoded prod LiveKit URL in .env.example)
- Hishab HF org: https://huggingface.co/hishab (44 models, 12 datasets)
- Mintlify docs: https://docs.verbex.ai (101 pages indexed in llms.txt)
- Our Kotha-side validation layer: `bangla-voice/platform/convex/flowgraph.ts`

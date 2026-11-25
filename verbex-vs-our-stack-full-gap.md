# Verbex vs Our Voice Stack - Complete Feature Gap Analysis

**Date:** 2026-11-25
**Our stack:** bdvoice-agent-platform-master (Rafiul's self-hosted Bangla voice agent)
**Reference:** Verbex AI by Hishab Singapore Pte Ltd (reverse-engineered)
**Purpose:** Full inventory of every feature gap, so planning is grounded in reality, not guesses

---

## TL;DR

We WIN on: **latency, cost, data residency, voice IP ownership, engineering rigor**.
We LOSE on: **~60 production features Verbex has, we don't yet**.

Realistic gap: **2+ years of product polish engineering** to match Verbex's full SaaS.
But: core voice loop is BETTER than Verbex today on the 5 things that matter for BD market.

---

## 1. Voice Engine Layer

### STT (Speech-to-Text)

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Primary STT | Deepgram Nova family (streaming) | bn-regional-ct2 (faster-whisper + Whisper medium fine-tune) | ✅ have |
| Multiple STT providers | Deepgram, OpenAI Whisper, Groq Whisper, Google STT, Azure STT, in-house Pia STT | Just bn-regional-ct2 + conformer-120m fallback | ❌ missing |
| Streaming interim transcripts | ✅ Yes (Deepgram streams) | ❌ No (Whisper can't stream) | ❌ MAJOR gap |
| Language detection auto | ✅ Yes | ❌ Pinned to Bangla | ❌ minor |
| English/Japanese support | ✅ Yes (3 languages total) | ❌ Bangla only | ❌ not needed for BD |
| Code-switching (bn+en in one sentence) | ✅ Partial | ❌ Not tuned | ⚠️ needed for BD informal speech |
| Confidence score per word | ✅ Yes | ❌ No | ❌ missing |
| Punctuation + capitalization | ✅ Yes | ❌ Basic | ⚠️ partial |
| Custom vocabulary | ✅ Yes (Deepgram keywords) | ❌ No | ❌ missing |

### LLM (Brain)

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Primary LLM | GPT-4.1 / Verbex Bangla Mini / choice of 11 models | Qwen3-27B self-hosted | ✅ have |
| LLM provider choice | 4 providers (OpenAI, Anthropic, Verbex, OpenAI-compatible) | Config switch + cloud fallback | ✅ mostly have |
| Realtime audio models (gpt-realtime, gemini-live) | ✅ Yes | ❌ No | ❌ missing |
| Multiple LLM models | 11+ options exposed | 1 primary + fallback | ⚠️ partial |
| Per-agent LLM override | ✅ Yes | ❌ No (global config) | ❌ missing |
| Streaming token output | ✅ Yes | ✅ Yes (OpenAI-compatible SSE) | ✅ have |
| Function calling | ✅ Yes (OpenAI format) | ❌ Not wired into agent | ❌ gap |
| Parallel tool calls | ✅ Yes | ❌ No | ❌ gap |
| Priority service tier (OpenAI) | ✅ Yes | ❌ No | ❌ nice-to-have |

### TTS (Mouth)

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Primary TTS | Pia/BnTTS (XTTS v2 continued-pretrain, 3.85k hr Bangla) | IndicF5 (F5-TTS flow matching) + reference clip | ✅ have |
| TTS providers | 8 options (Pia, OpenAI, ElevenLabs, Cartesia, Deepgram Aura, Azure, Inworld, Google) | Just IndicF5 + VITS fallback | ❌ minor |
| Named voice library | 18 voices (Maria, Rashida, Hena, Niloy, Rafiq, +13) | 1 reference-clip-based | ❌ MAJOR gap |
| Streaming TTS (sub-generation) | ✅ Yes (XTTS inference_stream) | ❌ No (F5-TTS is flow matching, cannot stream) | ❌ STRUCTURAL gap |
| Voice cloning (zero-shot from ref clip) | ❌ No (fixed library) | ✅ Yes (IndicF5 strength) | ✅ we WIN |
| Per-client custom voices | ❌ No (business model friction) | ✅ Potentially yes | ✅ we WIN unique |
| SSML support | ❌ Partial (prompt-level workarounds) | ❌ None | ⚠️ tied gap |
| Multi-language TTS | ✅ bn, en, ja | ❌ bn only | ❌ not needed for BD |
| Per-call playback-speed | ✅ Yes (agent-wide factor) | ✅ Yes (speed=1.15 config) | ✅ have |
| Pronunciations table | ✅ Yes `[{id, term, replacement}]` at agent level | ❌ Hardcoded in bangla_text.py | ⚠️ partial |
| Number normalization | ✅ Prompt-level | ✅ Pipeline-level (bangla_text.py) | ✅ we BETTER |
| Date normalization | ✅ Prompt-level | ✅ Partial in normalizer | ✅ mostly have |

### Phrase cache / Filler

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Pre-synthesized filler audio | ✅ Likely (tool_start_message etc) | ✅ Yes (phrase_cache.py, 2ms serve) | ✅ we have, BETTER docs |
| Voice fingerprinting on cache | ❓ Unknown | ✅ Yes (prevents stranger speaking) | ✅ we WIN |

---

## 2. Media & Transport Layer

| Feature | Verbex | Us | Status |
|---|---|---|---|
| WebRTC infrastructure | LiveKit Cloud (managed SFU) | ❌ None - plain HTTP file upload | ❌ MAJOR gap |
| Media server self-hostable | ✅ LiveKit open source | ❌ Not applicable | ❌ gap |
| SIP bridge for phone | ✅ LiveKit SIP + Twilio | ❌ None | ❌ MAJOR gap |
| PSTN phone number support | ✅ Twilio BD numbers (coming), US/JP | ❌ None (browser only) | ❌ MAJOR gap |
| Phone number import/porting | ✅ Yes (import_numbers API) | ❌ No | ❌ gap |
| Multi-region media POP | ✅ Chicago, with plans for more | ❌ Single location | ❌ scaling gap |
| Audio codec: Opus/RED | ✅ Yes (WebRTC standard) | ❌ WebM/Opus posted as file | ❌ gap |
| Low-latency streaming audio | ✅ Yes (true WebRTC) | ❌ Whole-file transport | ❌ MAJOR gap |
| Real-time barge-in | ✅ Yes | ❌ Approximate only (file already sent) | ❌ MAJOR gap |
| Outbound call API | ✅ Yes (dial-outbound-phone-call) | ❌ No | ❌ gap |
| Batch outbound dialing | ✅ Yes (batch_call) | ❌ No | ❌ gap |
| Call recording (room-level) | ✅ LiveKit Egress to MP3/MP4 | ❌ No | ❌ gap |

---

## 3. Agent Types & Flow Builder

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Simple Agent (one prompt + LLM) | ✅ Yes | ✅ Yes | ✅ have |
| Agentic Agent (tool-calling) | ✅ Yes (default GPT-4o) | ❌ Not wired | ❌ gap |
| Flow Agent (visual builder) | ✅ Yes, 11 node types | ❌ Not built | ❌ MAJOR gap |
| Flow node: start | ✅ | ❌ | ❌ |
| Flow node: conversation (free LLM chat) | ✅ (modes: full/static-only/prompt-only) | ❌ | ❌ |
| Flow node: collection (slot filling) | ✅ | ❌ | ❌ |
| Flow node: variable-extraction | ✅ | ❌ | ❌ |
| Flow node: math-operation | ✅ (add/sub/mul/div/mod/set) | ❌ | ❌ |
| Flow node: call-api | ✅ | ❌ | ❌ |
| Flow node: decision (branching) | ✅ | ❌ | ❌ |
| Flow node: transfer (cold/warm-non-agentic/warm-agentic) | ✅ | ❌ | ❌ |
| Flow node: sms | ✅ (Twilio integration) | ❌ | ❌ |
| Flow node: press-digit (DTMF) | ✅ | ❌ | ❌ |
| Flow node: end-call | ✅ | ❌ | ❌ |
| Visual flow editor UI | ✅ Yes (drag-drop graph) | ❌ None | ❌ MAJOR gap |
| Flow edge types (7 kinds) | ✅ session-start, direct-edge, conditional-edge, on-reply-edge, logic-branch, logic-default, error-fallback | ❌ | ❌ |
| Per-node LLM/STT/TTS override | ✅ Yes | ❌ No | ❌ gap |
| Per-node voice settings | ✅ `{source: "flow-default"}` placeholders | ❌ | ❌ gap |
| Flow validation Layer 2 | ❌ Verbex accepts broken graphs (we have to add this ourselves) | N/A | ⚠️ not applicable yet |
| Realtime Agent mode | ✅ gpt-realtime / gemini-live | ❌ No | ❌ gap |

---

## 4. Tools & Integrations

### Built-in tools

| Tool | Verbex | Us | Status |
|---|---|---|---|
| Send Email (Gmail) | ✅ Yes | ❌ No | ❌ gap |
| Call Transfer (cold) | ✅ Yes | ❌ No | ❌ gap |
| Call Transfer (warm, with whisper) | ✅ Yes | ❌ No | ❌ gap |
| Call Transfer (warm-agentic) | ✅ Yes | ❌ No | ❌ gap |
| Calendar (Google appointment booking) | ✅ Yes | ❌ No | ❌ gap |
| SMS send | ✅ Yes (Twilio) | ❌ No | ❌ gap |

### Custom tools

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Custom HTTP tool | ✅ OpenAI function-calling schema | ❌ Not wired | ❌ MAJOR gap |
| Tool definition API | ✅ `/v2/ai-agents/{id}/custom-tools/` | ❌ No | ❌ gap |
| Tool lifecycle messages | ✅ start/delayed/complete/failed (spoken by TTS) | ❌ No | ❌ gap |
| Tool timeout handling | ✅ Yes | ❌ No | ❌ gap |
| Make.com webhook integration | ✅ Yes | ❌ No | ❌ gap |
| Zapier webhook integration | ✅ Yes | ❌ No | ❌ gap |
| Tool execution logs | ✅ Yes | ❌ No | ❌ gap |
| Tool success/failure tracking | ✅ Yes (per-tool rates in analytics) | ❌ No | ❌ gap |

---

## 5. Knowledge Base (RAG)

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Vector database | Pinecone (default) + 4 alternatives (Qdrant, Weaviate, Milvus, pgvector) | ❌ None | ❌ MAJOR gap |
| Embedding model | text-embedding-3-large (default) + text-embedding-3-small | ❌ None (README says "blocked: needs embedding model") | ❌ gap |
| Chunking strategies | 5 (fixed/recursive/semantic/sentence/paragraph) | ❌ None | ❌ gap |
| Chunk size/overlap config | ✅ Default 512/50 | ❌ N/A | ❌ |
| Hybrid retrieval (vector + keyword) | ✅ Yes | ❌ No | ❌ gap |
| File upload: PDF | ✅ Yes | ❌ No | ❌ gap |
| File upload: DOCX | ✅ Yes | ❌ No | ❌ gap |
| File upload: TXT | ✅ Yes | ❌ No | ❌ gap |
| File upload: MD | ✅ Yes | ❌ No | ❌ gap |
| File upload: XLSX/XLS | ✅ Yes | ❌ No | ❌ gap |
| Website crawl | ✅ Yes | ❌ No | ❌ gap |
| Async ingestion | ✅ 202 Accepted + poll status | ❌ No | ❌ gap |
| Document status tracking | ✅ Yes | ❌ No | ❌ gap |
| KB per agent | ✅ Yes | ❌ No | ❌ gap |
| Hot knowledge updates | ❌ Delete + resubmit (not atomic) | ❌ N/A | ⚠️ both weak |

---

## 6. Call Management & Observability

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Call log persistent storage | ✅ MongoDB ObjectIds | ❌ Dict in memory (lost on restart) | ❌ MAJOR gap |
| Call recording (audio file saved) | ✅ Yes (signed URL via Egress) | ❌ No | ❌ MAJOR gap |
| Call recording playback | ✅ Yes | ❌ Only last 8 turns in memory | ❌ gap |
| Transcript persistent storage | ✅ Yes | ❌ No | ❌ MAJOR gap |
| Post-call analysis | ✅ Configurable, extracts sentiment/intent/outcomes/custom | ❌ No | ❌ MAJOR gap |
| Sentiment extraction | ✅ Yes | ❌ No | ❌ gap |
| Intent classification | ✅ Yes | ❌ No | ❌ gap |
| Custom data extraction (per agent) | ✅ Yes | ❌ No | ❌ gap |
| Call duration tracking | ✅ Yes | ✅ Yes (in-memory) | ⚠️ partial |
| Call status tracking (registered/ongoing/ended) | ✅ Yes | ❌ No | ❌ gap |
| Disconnect reason tracking | ✅ Yes | ❌ No | ❌ gap |
| Knowledge miss logging | ✅ Yes (questions docs couldn't answer) | ❌ No | ❌ gap |
| Per-call metadata (custom fields) | ✅ Yes | ❌ No | ❌ gap |

---

## 7. API & Developer Experience

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Public REST API | ✅ Full (api.verbex.ai) | ❌ 14 internal test routes only | ❌ MAJOR gap |
| OpenAPI specification | ✅ Yes | ❌ No | ❌ gap |
| API versioning (v1, v2) | ✅ Yes | ❌ No | ❌ gap |
| Bearer token auth | ✅ Yes | ❌ No | ❌ gap |
| API key management UI | ✅ Yes | ❌ No | ❌ gap |
| Multi-tenant (org_id injection) | ✅ Yes (`x-user-org-id` + `x-verbex-id`) | ❌ Single admin | ❌ MAJOR gap |
| Rate limiting | ✅ Yes (WAF + API gateway) | ❌ No | ❌ gap |
| Request/response logs | ✅ Yes | ❌ No | ❌ gap |
| SDKs (JS) | ✅ `@verbex-ai/verbex-js-sdk@1.0.0` | ❌ No | ❌ gap |
| SDKs (Python) | ✅ Yes | ❌ No | ❌ gap |
| SDKs (Java) | ✅ Yes | ❌ No | ❌ gap |
| SDKs (React Native) | ✅ Yes | ❌ No | ❌ gap |
| Public API documentation | ✅ Mintlify (101 pages) | ❌ No | ❌ gap |
| API staging environment | ✅ `api-stage.verbex.ai` | ❌ No | ❌ gap |

---

## 8. Webhooks & Events

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Webhook subscription API | ✅ Yes | ❌ No | ❌ gap |
| Call started event | ✅ Yes | ❌ No | ❌ gap |
| Call ended event | ✅ Yes | ❌ No | ❌ gap |
| Call failed event | ✅ Yes | ❌ No | ❌ gap |
| Transcript event | ✅ Yes | ❌ No | ❌ gap |
| Tool invocation event | ✅ Yes | ❌ No | ❌ gap |
| HMAC-SHA256 signed payloads | ✅ Yes (`"verbex-webhook-v1"` key derivation) | ❌ No | ❌ gap |
| Webhook retry logic | ✅ Yes | ❌ No | ❌ gap |
| Webhook delivery logs | ✅ Yes | ❌ No | ❌ gap |

---

## 9. Auth, Multi-tenancy & Security

| Feature | Verbex | Us | Status |
|---|---|---|---|
| OAuth2 / OIDC login | ✅ Auth0 Japan tenant | ❌ Basic auth | ❌ MAJOR gap |
| Workspaces / organizations | ✅ Yes | ❌ No | ❌ MAJOR gap |
| RBAC (Admin/Member/Viewer) | ✅ Yes | ❌ No | ❌ gap |
| Per-agent permissions | ✅ Yes | ❌ No | ❌ gap |
| Per-KB permissions | ✅ Yes | ❌ No | ❌ gap |
| User management UI | ✅ Yes | ❌ No | ❌ gap |
| Invite users | ✅ Yes | ❌ No | ❌ gap |
| Tenant data isolation | ✅ Yes (404 on cross-tenant, no 403 leakage) | ❌ N/A | ❌ gap |
| WAF (web application firewall) | ✅ Yes (Cloudflare-style) | ❌ No | ❌ gap |
| IP allowlist | ✅ Yes | ❌ No | ❌ gap |
| Bot filtering | ✅ Yes | ❌ No | ❌ gap |
| KMS encryption at rest | ✅ Yes (secrets, Twilio creds, tool auth) | ❌ Plain .env | ❌ gap |
| Audit logs | ✅ Yes | ❌ No | ❌ gap |
| Session management | ✅ Yes (Auth0) | ❌ No | ❌ gap |
| 2FA/MFA | ✅ Yes (via Auth0) | ❌ No | ❌ gap |
| SOC2 / compliance posture | ⚠️ Not stated but enterprise clients suggest yes | ❌ No | ❌ gap |

---

## 10. Advanced Voice Tuning

| Feature | Verbex | Us | Status |
|---|---|---|---|
| VAD activation threshold | ✅ Configurable | ✅ Config | ✅ have |
| Endpointing delay | ✅ `max_endpointing_delay` | ✅ `wait_time_before_detecting_end_of_speech` | ✅ have |
| Minimum speech duration | ✅ Yes | ✅ Yes | ✅ have |
| Interruption detection | ✅ Yes | ⚠️ Partial | ⚠️ gap |
| Barge-in handling | ✅ Yes | ⚠️ Approximate only | ❌ gap |
| Ambient sound volume | ✅ Configurable | ❌ No | ❌ gap |
| AMD (voicemail detection) | ✅ Yes (`amd_voicemail_message`, `voicemail_detection_timeout_seconds`) | ❌ No | ❌ gap |
| Voicemail message playback | ✅ Yes | ❌ No | ❌ gap |
| Pronunciations table | ✅ Agent-level `[{id, term, replacement}]` | ⚠️ Hardcoded in normalizer | ⚠️ partial |
| Dynamic data config | ✅ `dynamic_data_config` per call | ❌ No | ❌ gap |
| End call instruction | ✅ `end_call_instruction` | ❌ No | ❌ gap |
| Agent personality tuning | ✅ Multiple fields | ⚠️ Prompt-only | ⚠️ partial |

---

## 11. Dashboard & UI

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Web dashboard | ✅ console.verbex.ai (Next.js) | ❌ Local test console only (localhost:8305) | ❌ MAJOR gap |
| Dark mode | ✅ Yes | ❌ No | ❌ nice-to-have |
| Agent list view | ✅ Yes | ❌ No | ❌ gap |
| Agent create/edit UI | ✅ Yes | ❌ Config file only | ❌ gap |
| Agent version history | ✅ Yes | ❌ No | ❌ gap |
| Version rollback UI | ✅ Yes | ❌ No | ❌ gap |
| Call history page | ✅ Yes | ❌ Last 8 in memory | ❌ gap |
| Transcript viewer | ✅ Yes | ❌ No | ❌ gap |
| Audio playback in dashboard | ✅ Yes | ❌ No | ❌ gap |
| Analytics / metrics page | ✅ Yes (completion rate, duration, sentiment mix) | ❌ Basic stats | ❌ gap |
| Live monitoring (active calls) | ✅ Yes | ❌ No | ❌ gap |
| Agent testing in dashboard | ✅ Yes (built-in test caller) | ✅ Partial (console.py) | ⚠️ partial |
| Public demo widget | ✅ Yes (homepage iframe) | ❌ No | ❌ gap |
| Multi-language UI | ✅ English | ❌ English (console.py) | ⚠️ tied |

---

## 12. Billing & Pricing Infrastructure

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Multi-tier pricing | ✅ Yes | ❌ No | ❌ gap |
| Per-minute metered billing | ✅ Yes | ❌ No | ❌ gap |
| Usage dashboard | ✅ Yes | ❌ No | ❌ gap |
| Invoice generation | ✅ Yes | ❌ No | ❌ gap |
| Credit card payment | ✅ Yes (Stripe) | ❌ No | ❌ gap |
| bKash/Nagad payment (for BD) | ❌ No (Stripe USD card only) | ❌ No | ⚠️ both gap, we need for BD |
| Enterprise contracts | ✅ Yes | ❌ No | ❌ gap |
| Trial period handling | ✅ Yes | ❌ No | ❌ gap |
| Overage charges | ✅ Yes | ❌ No | ❌ gap |
| Agent-level cost attribution | ✅ Yes | ❌ No | ❌ gap |

---

## 13. Customer Templates & Onboarding

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Industry templates (BD-specific) | ❌ Generic US templates | ❌ Config example only | ⚠️ both need |
| Template library | ✅ Yes (US-centric) | ❌ No | ❌ gap |
| Guided onboarding wizard | ✅ Yes | ❌ No | ❌ gap |
| Prompt authoring help | ❌ Minimal | ❌ None | ⚠️ tied |
| Bangla prompt linter | ❌ No | ❌ No | ⚠️ opportunity for us |
| Test caller UI | ✅ Yes | ✅ Yes (console.py) | ✅ tied |
| Published test URL | ✅ Yes | ❌ No | ❌ gap |

---

## 14. Infrastructure & Reliability

| Feature | Verbex | Us | Status |
|---|---|---|---|
| 24/7 SLA | ✅ Enterprise contracts (KDDI, Hitachi) | ❌ No | ❌ MAJOR gap |
| Multi-region failover | ✅ Partial (Chicago + planned) | ❌ Single box | ❌ gap |
| Horizontal scaling | ✅ Yes (LiveKit worker pool) | ❌ max_concurrency=2 | ❌ MAJOR gap |
| Load balancer | ✅ Yes | ❌ No | ❌ gap |
| Auto-recovery on failure | ✅ Yes | ❌ Manual | ❌ gap |
| Database replication | ✅ Yes (MongoDB) | ❌ N/A (no DB) | ❌ gap |
| Backup & restore | ✅ Yes | ❌ No | ❌ gap |
| Monitoring + alerting | ✅ Yes (internal) | ❌ Basic prints | ❌ gap |
| Status page (public) | ⚠️ Likely (enterprise) | ❌ No | ❌ gap |
| Multiple brain workers | ✅ Yes | ❌ max_concurrency=2 only | ❌ MAJOR gap |

---

## 15. Compliance & Data

| Feature | Verbex | Us | Status |
|---|---|---|---|
| Data residency (BD) | ❌ Chicago-hosted | ✅ BD-local | ✅ we WIN BIG |
| GDPR controls | ✅ Implied (EU enterprise) | ❌ None | ❌ gap (if selling abroad) |
| Data retention policies | ✅ Yes | ❌ None | ❌ gap |
| Right to delete | ✅ Yes | ❌ No | ❌ gap |
| PII redaction | ✅ Possible via tools | ❌ No | ❌ gap |
| HIPAA posture | ⚠️ Not stated | ❌ No | ❌ gap (medical use) |
| BTRC compliance (BD regulatory) | ⚠️ Not applicable (foreign) | ⚠️ Possible with local SIP + colo | ✅ achievable, Verbex can't |
| Banking regulatory (Bangladesh Bank) | ❌ Can't comply (Chicago hosting) | ✅ Possible | ✅ we WIN |

---

## 16. What WE have that Verbex DOESN'T

These are OUR genuine advantages, not just features we're catching up on.

| Feature | Us | Verbex |
|---|---|---|
| Measured latency (real calls) | ✅ 1,012-1,864ms (own measurement) | ❌ 2,235-2,515ms (from their own recordings) |
| 100% BD data residency | ✅ Yes | ❌ No |
| Own fine-tuned voice (IndicF5 + reference) | ✅ Yes, portable | ❌ Locked to platform |
| Per-client custom voice clone (business model enables) | ✅ Possible | ❌ Business model friction |
| Open source LLM runtime (llama.cpp) | ✅ Yes | ❌ Closed |
| Self-hostable end-to-end | ✅ Yes | ❌ Only their cloud |
| No vendor lock-in | ✅ Yes | ❌ Yes lock-in |
| Rigorous measurement culture (BUILD-CHECK.md) | ✅ Yes, visible in repo | ❌ Can't verify |
| Honest README listing "what is NOT done" | ✅ Yes | ❌ Marketing-only |
| EPSS schedule for F5-TTS step pruning | ✅ Implemented with toggle | N/A |
| Phrase cache with voice fingerprinting | ✅ Yes | ⚠️ Unknown |
| Brain keepalive thread with backoff | ✅ Yes | N/A (they don't have cold start problem) |
| Config-file-driven agents (vs forks) | ✅ Yes | ✅ Yes (both have this) |
| Boot-time config validation | ✅ Yes (refuses typos) | ✅ Yes (both) |
| Cloud fallback with visibility | ✅ Yes (reports which brain answered) | ❌ No visibility |

---

## 17. Realistic Priority Order to Close the Gap

Not everything matters equally. Ranked by business impact.

### Critical (blocking first production customer):
1. **Call log + recording persistent storage** (MongoDB or Convex)
2. **Public API with tenant isolation**
3. **RBAC / multi-tenancy**
4. **Tool calling (at minimum: HTTP tool with OpenAI function schema)**
5. **Phone/SIP integration (via LiveKit)**

### High (first 10 customers):
6. **Knowledge base + RAG** (Convex + pgvector or Qdrant)
7. **Webhook subscription API** (HMAC signed)
8. **Billing infrastructure** (per-minute metered, bKash/Nagad for BD)
9. **Dashboard with call history, transcripts, playback**
10. **Post-call analysis** (sentiment, intent, custom extraction)

### Medium (scaling to 100 customers):
11. **Flow Agent visual builder** (11 node types)
12. **Horizontal brain scaling** (multiple workers, load balancer)
13. **Multi-tier pricing + usage dashboard**
14. **Industry templates** (BD shop, clinic, restaurant, bank-specific)
15. **Agent version history + rollback**
16. **Real barge-in (via LiveKit streaming transport)**

### Lower (differentiators later):
17. **Per-client custom voice cloning as feature** (our moat)
18. **Realtime models support** (gpt-realtime, gemini-live)
19. **Multiple STT providers** (fallback options)
20. **Multi-language (en, hi) support**
21. **Advanced voicemail (AMD) handling**
22. **SOC2 compliance posture** (enterprise unlock)

### Nice-to-have (polish):
23. SDKs (JS, Python)
24. Public documentation site (Mintlify or similar)
25. Status page
26. Public demo widget

---

## 18. Realistic Timeline

### Phase 1: Production-ready MVP (Months 1-3, ~15-20 lakh BDT)
- Items 1-5 above (call log, API, RBAC, tool calling, phone)
- Enables first paying client (bank or SME shop)

### Phase 2: First 10 customers (Months 3-6, ~10-15 lakh BDT)
- Items 6-10 (RAG, webhooks, billing, dashboard, analytics)
- Pays for itself with revenue

### Phase 3: Scale to 100 customers (Months 6-12, ~15-20 lakh BDT)
- Items 11-16 (flow builder, scaling, templates)
- Self-service onboarding

### Phase 4: Differentiator moats (Year 2, ~20-30 lakh BDT)
- Items 17-26
- Enterprise + international expansion

**Total engineering investment to match Verbex: ~60-85 lakh BDT over 24 months, 2-3 engineers**

**Vs Verbex's investment**: 10 years of Hishab Singapore + tens of crores. We're not matching that budget, we're being surgical about what matters for BD market.

---

## 19. Bottom Line

### Are we as good as Verbex today?
**No.** Verbex has 60+ production features we lack. 2+ years of polish.

### Are we better than Verbex on what matters for BD market?
**Yes.** Latency, cost, data residency, voice IP ownership.

### Can we catch up?
**Partially.** Features yes. 10-year Bangla NLP ecosystem knowledge no.

### Do we need to catch up on everything?
**No.** Focus on BD market needs:
- Phone/SIP (bank + SME both need)
- Call log + analytics (any serious client)
- Billing + bKash/Nagad (BD-specific, Verbex can't)
- RAG + tool calling (SME use cases)
- Dashboard (self-service)

Skip for now:
- Multi-region media (BD-focused for 2 years)
- Multi-language (bn only)
- SOC2 (until enterprise)
- 18 voices (per-client custom is better moat)

### The thesis:
**Compete on what Verbex can't easily match (local cost, latency, compliance), not on what they can out-engineer us on (features).**

---

## Sources

- Verbex architecture reverse-engineering: `verbex-voice-agent-architecture.md` in this repo
- Verbex full reference: `/Users/rafiulislam/Work/n8n/bangla-voice/VERBEX-ARCHITECTURE.md` (613 lines)
- Our stack: `bdvoice-agent-platform-master/` README.md + BUILD-CHECK.md
- Kotha platform plan: `/Users/rafiulislam/Work/n8n/bangla-voice/platform/docs/BD-ADAPTATION.md`

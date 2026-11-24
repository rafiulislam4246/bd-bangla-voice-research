# LiveKit - What It Is and Why Voice Agents Use It

Date: 2026-11-24
Context: Verbex, Retell, Vapi, Pipecat, Daily.co all run on LiveKit. We currently run HTTP microservices (ears.py 8300 + mouth.py 8301 + try.py 8302) which is effectively "reinvented LiveKit, worse." This doc explains what LiveKit gives and what we lose by not using it.

---

## What LiveKit is

**Two products stacked:**

1. **LiveKit SFU** - a WebRTC Selective Forwarding Unit. Routes audio/video between participants in a "room". Handles NAT traversal, packet-loss recovery, codec negotiation, bandwidth adaptation. Written in Go. Apache-2.0 open source. Cloud-hosted or self-hostable.

2. **LiveKit Agents (Python SDK)** - a framework for building voice/video AI agents that join rooms as participants. Wraps the state machine (listening → thinking → speaking), audio frame handling, STT/LLM/TTS plugin pattern.

Together: the complete voice agent stack from carrier to AI model.

---

## Why Verbex (and every competitor) uses it

### 1. Phone + browser callers use the SAME code path
```
PSTN caller → Twilio SIP trunk → LiveKit SIP bridge → LiveKit room
Browser caller → WebRTC → LiveKit room
```
Agent joins the room. It does not know whether the caller is on phone or browser - same code handles both.

**Without LiveKit**: build separate pipelines for SIP audio (RTP, G.711) and browser audio (Opus, WebRTC). Months of engineering, all redundant.

### 2. WebRTC complexity is already solved
Media plumbing LiveKit handles:
- ICE (NAT traversal across firewalls)
- STUN / TURN (relay servers for strict networks)
- DTLS-SRTP (encrypted media)
- Codec negotiation (Opus, G.711, G.722)
- Jitter buffering (variable network delay compensation)
- Packet loss recovery via RED (redundant encoding)
- Bandwidth adaptation (bitrate up/down per network conditions)
- Simulcast + SVC (multi-quality streams)

**Our current stack**: none of this. We assume a WAV file lands on `/listen` endpoint. Real phone calls don't work that way - they stream continuously over packet networks.

### 3. Agent joins room like any participant
Mental model:
- Room = conversation
- User = participant 1 (audio stream)
- Agent = participant 2 (audio stream, published by Python process)
- Agent subscribes to user's audio → processes → publishes synthesized audio back

**Abstraction clean**: agent code sees audio frames and publishes audio frames. No "which codec, which transport, which protocol."

### 4. SIP bridge built-in = phone calls in one config line
Twilio Elastic SIP Trunk setup:
- Termination URI: `sip:<your-project>.sip.livekit.cloud`
- Assign Bangladeshi number to the trunk
- Done

**Without LiveKit**: build SIP server (PJSIP, Kamailio, FreeSWITCH), handle RTP transcoding, SIP registrar. Weeks of expertise.

### 5. STT / LLM / TTS as plugins
```python
from livekit.agents import AgentSession
from livekit.plugins import deepgram, openai, elevenlabs

session = AgentSession(
    stt=deepgram.STT(language="bn"),
    llm=openai.LLM(model="gpt-4.1-mini"),
    tts=elevenlabs.TTS(voice_id="..."),
)
```
Standard interface. Swap any component:
- `deepgram.STT()` → `WhisperSTT()` (custom plugin wrapping our bn-regional-ct2)
- `elevenlabs.TTS()` → `F5TTS()` (custom plugin wrapping our mouth.py)
- Pattern 2 (in-process pipeline) unlocked - no HTTP between stages

### 6. State machine auto-managed
LiveKit emits:
- `lk.agent.state = listening` when VAD detects speech start
- `lk.agent.state = thinking` when LLM is generating
- `lk.agent.state = speaking` when TTS is outputting

Frontend can subscribe to these events for visual feedback (waveform animation, "AI is thinking..." indicator). Verbex uses this.

### 7. Horizontal scaling
Multiple agent workers register with LiveKit. New call → LiveKit picks a worker, dispatches the room. Idle workers consume zero resources. Scale by running more workers.

**Our current setup**: single Python process per service, no pool, no dispatch.

### 8. Recording via Egress
LiveKit Egress service records rooms to MP3 / MP4 / HLS. No code - just REST config. Signed URL served from S3/GCS bucket.

**Without LiveKit**: self-record audio frames, encode, upload, serve - all custom.

### 9. Industry standard - not a weird pick
Voice agent companies using LiveKit:
- **Retell AI** (voice API startup)
- **Vapi** (voice API startup)
- **Verbex** (what we're reverse-engineering)
- **Pipecat** (open source voice framework by Daily.co)
- **Daily.co Prebuilt** (their own product)

Alternatives and why they lost mindshare:
- **mediasoup** (Node.js SFU) - lower level, no SIP, no Agents SDK, DIY everything
- **Janus** - C-based, SIP support, no native Agents framework, harder plugin pattern
- **Jitsi** - built for video conferencing not voice agents, heavier
- **Kurento** - deprecated-ish, Java-heavy, SIP works but awkward
- **Twilio Media Streams** - proprietary, raw WebSocket audio only, no agent framework

### 10. Cloud or self-host, same API
- LiveKit Cloud free tier: ~10,000 participant-minutes/month
- Scale pricing: ~$0.005 per participant-minute
- Self-host: Apache-2.0, free forever, run on own Kubernetes
- Switch between cloud/self-host by changing one URL

### 11. Open source, no vendor lock
Apache-2.0 license. All code at github.com/livekit/*. Switch vendor if LiveKit Cloud raises prices - self-host the same binaries, same SDK.

---

## What we lose by NOT using it (current setup assessment)

Our bdvoice-stt-tts-master ships with:
- `ears.py` on port 8300 - FastAPI HTTP endpoint
- `mouth.py` on port 8301 - FastAPI HTTP endpoint
- `try.py` on port 8302 - frontend page

### Phone calls
❌ IMPOSSIBLE in current form. No SIP ingestion. To add phone support without LiveKit, we'd build:
- SIP server (PJSIP/FreeSWITCH)
- RTP media handler
- G.711 ↔ our pipeline codec bridging
- Twilio trunk integration

Weeks of work, parallel to the actual AI work.

### Browser calls
⚠️ Partial. `try.py` probably posts WAV files. True real-time WebRTC from browser requires:
- Signaling server
- WebRTC peer connection code
- ICE/TURN handling
- Opus encoding client-side

Roll our own ≈ months. LiveKit = `livekit-client-js` SDK, 10 lines of frontend code.

### Scaling
❌ Single process bottleneck. Concurrent callers queue through one FastAPI instance.

### In-process STT→LLM→TTS pipeline
❌ HTTP serialization between stages adds 40-100ms per call. LiveKit Agents = direct Python function calls.

### State machine
❌ No "listening/thinking/speaking" events published to frontend. Users see no feedback during the 1500ms pipeline cost.

### Recording
❌ Self-implement audio capture + encode + upload. Not hard but 1-2 days work.

---

## Migration path for us (practical)

### Phase A: Local prototype with LiveKit (1-2 days)
1. `pip install livekit-agents livekit-plugins-openai`
2. Write `agent.py` wrapping our ears.py + mouth.py as custom plugins:
   ```python
   class BnRegionalWhisperSTT(STT):
       def __init__(self):
           self.model = ... # import ears.py logic
       async def _recognize_impl(self, buffer: AudioBuffer) -> SpeechEvent:
           return await ears_logic(buffer)

   class IndicF5TTS(TTS):
       def __init__(self):
           self.model = ... # import mouth.py logic
       async def _synthesize_impl(self, text: str) -> SynthesisEvent:
           return await mouth_logic(text)
   ```
3. Wire into `AgentSession` with chosen LLM
4. Test locally in browser via `livekit-client-js`

### Phase B: Phone number integration (2-3 days additional)
1. Buy Twilio BD number (if available) or Bangladeshi carrier via LiveKit SIP
2. Configure SIP trunk: `sip:<project>.sip.livekit.cloud`
3. Test inbound: real phone → agent answers

### Phase C: Streaming patterns (from pipeline-streaming-architecture.md)
1. Pattern 2 (in-process): automatically unlocked by LiveKit Agents
2. Pattern 3 (brain token streaming): enable `stream=True` on LLM plugin
3. Pattern 4 (TTS sentence boundary): write custom TTS plugin dispatching per sentence

### Phase D: Scale (if traffic demands)
- Run multiple agent workers
- LiveKit dispatches per room
- Horizontal scale, no code change

---

## Trade-offs (be honest)

### Downsides of adopting LiveKit
1. **Added dependency** - 5MB Python package, more moving parts
2. **Learning curve** - Agents SDK conventions, room concepts, participant lifecycle (1-2 days to grok)
3. **Cloud cost at scale** - $0.005/participant-min adds up (10000 min = $50/mo). Self-host eliminates.
4. **Overkill for batch TTS/STT** - if we just transcribe uploaded files, FastAPI is simpler
5. **Reference docs assume you know WebRTC concepts** - participant, track, publication, subscription

### When to NOT use LiveKit
- Batch audio transcription (just HTTP API is fine)
- TTS used as a text-to-file service (just synthesize + save)
- Internal tooling with no live callers

### When LiveKit is non-negotiable
- Phone call support (SIP)
- Real-time browser voice (true WebRTC, not WebSocket audio chunks)
- Multiple concurrent callers at scale
- Verbex/Retell/Vapi-class voice agent product

---

## For our specific project

Current `bdvoice-stt-tts-master` is **positioned as a voice agent** (replacing an 8 taka/min service). That means phone calls are the eventual target. Browser-only use is a phase 0 testing tool.

**Recommendation**: adopt LiveKit at Phase A (1-2 days work). It unblocks:
- Phone call support
- Scaling
- In-process pipeline (Pattern 2 from streaming doc)
- Standard framework others can read (hiring, open-sourcing)

Not adopting = permanent ceiling on what this service can do.

---

## Sources

- LiveKit main site: https://livekit.io
- LiveKit Agents docs: https://docs.livekit.io/agents/
- LiveKit SIP: https://docs.livekit.io/sip/
- GitHub: https://github.com/livekit/livekit (SFU, Go, Apache-2.0)
- Agents Python SDK: https://github.com/livekit/agents
- Cloud regions: https://docs.livekit.io/home/cloud/regions
- Verbex uses LiveKit confirmed in `verbex-voice-agent-architecture.md`

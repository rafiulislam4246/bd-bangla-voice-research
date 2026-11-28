# Claude Execution Guide: Deploy Bangla Voice Agent on Olares One

**AUDIENCE**: Another Claude instance (Anthropic AI assistant) running on another laptop.
**MISSION**: Deploy a 3-service Bangla voice agent stack onto user's Olares One hardware box.
**CONTEXT**: User is Rafiul Islam. Reply to him in Banglish (Latin-script Bangla).
**WRITTEN**: 2026-11-28
**PRIMARY REFERENCE**: https://github.com/rafiulislam4246/bd-bangla-voice-research/blob/main/olares-deployment-plan.md

---

## 0. WHO/WHAT/WHY (so you understand what you're doing)

### Who is the user
- **Rafiul Islam** (`mdrafiulislam4246@gmail.com`)
- Builds BD voice agent platform "Kotha" for selling voice AI to BD SMEs
- Current voice product: `bdvoice-agent-platform-master` (in `~/Downloads/bdvoice-agent-platform-master.zip`)
- Pays Verbex 8 BDT/min currently, wants to replace with own stack

### What you're deploying
A production voice agent with 3 services sharing one GPU:

```
┌─────────────────────────────────────┐
│  caller (phone/browser)             │
│   ↓ audio                           │
├─────────────────────────────────────┤
│  EAR: bn-regional-ct2 STT  (2GB)   │  <- Bangla transcription
│   ↓ text                            │
├─────────────────────────────────────┤
│  BRAIN: Qwen2.5-7B Q6 LLM (6GB)   │  <- Bangla understanding
│   ↓ response text                   │
├─────────────────────────────────────┤
│  MOUTH: IndicF5 TTS (3GB)          │  <- Bangla voice synthesis
│   ↓ audio                           │
└─────────────────────────────────────┘
```

### Why this plan exists
Current stack (RTX 4070 laptop + remote brain box) latency: 1,914ms. Target <1,500ms.
Olares One (RTX 5090 Mobile 24GB) can run all 3 services on one box. Projected: 650-950ms.

### Hardware target
- **Olares One** (a physical box in user's office, he has access)
- **GPU**: RTX 5090 Mobile 24GB VRAM (Blackwell architecture)
- **CPU**: Intel Core Ultra 9 275HX
- **RAM**: 96GB DDR5
- **Storage**: 2TB NVMe
- **OS**: Olares OS (Linux-based personal cloud OS, uses k3s under the hood)
- **CRITICAL**: 8GB VRAM reserved for other project. You have **16GB budget**.

---

## 1. PREREQUISITES (what must be true before you start)

### On user's machine (yours)
- [ ] You have terminal access via Bash tool
- [ ] User has internet connection
- [ ] User can SSH into Olares One, OR run commands locally if your terminal reaches it

### On Olares One (user verifies)
- [ ] Olares version 1.12.6 or newer (`olares-cli version`)
- [ ] NVIDIA driver 570+ for Blackwell support
- [ ] CUDA 12.8+ (nvidia-smi output)
- [ ] `nvidia-container-toolkit` installed and working
- [ ] k3s running with GPU scheduler
- [ ] At least 25GB free disk space
- [ ] Other 8GB project is IDLE during initial install (to avoid GPU scheduling fights)

### Reference files user has
- `~/Downloads/bdvoice-agent-platform-master.zip` - working voice agent source code
- `~/Work/tts-stt-research/` - our research notes
- `~/Work/tts-stt-research/olares-deployment-plan.md` - prior planning (predecessor to this guide)

### Research already done (don't redo these)
- Olares GPU support: CONFIRMED (version 1.12.6)
- CUDA version: must be 12.8+ for Blackwell (driver 595.84 blocks 13.3)
- GPU sharing mechanism: time-slicing (no memory isolation)
- Deploy pattern: 3 separate Olares apps, not multi-container
- Cold-start order: brain → mouth → ear with 30s gaps

---

## 2. CRITICAL RULES (do not violate, per user's memory system)

### Must-follow
1. **Reply in Banglish** (Latin-script Bangla) - user prefers this
2. **Never claim 100% ready** - always "proven on prod" vs "not proven"
3. **Verify before done** - HTTP 200 is NOT proof, actual voice turn is
4. **No destructive git/deploy without explicit user permission** - including setting workflows active
5. **Never override git author with `-c` flags** - global config is correct
6. **No em-dashes** in output
7. **Short answers, full substance** - no essays
8. **Batch fix, test local, deploy once** - don't half-fix
9. **Regression check after every fix** - quote numbers
10. **Check central store first** - `e:\N8N\api-keys.env` for secrets if needed

### Never-do
- Don't run paid API calls without explicit permission ($3 burn rule)
- Don't claim things work without evidence
- Don't push git commits without user approval
- Don't try to fix things you don't understand

---

## 3. EXECUTION PHASES

### Phase 0: Environment verification (10 min)

**Goal**: Confirm Olares One is in correct state before touching anything.

**Have user run** (paste output back to you):

```bash
# GPU check
nvidia-smi
# Expected: "NVIDIA GeForce RTX 5090 Mobile", 24576 MiB, driver ≥570, CUDA ≥12.8

# Olares version
olares-cli version
# Expected: 1.12.6 or newer

# Container runtime
nvidia-container-cli info | head -20
# Expected: shows GPU details, no errors

# k3s nodes
kubectl get nodes -o wide
kubectl describe nodes | grep -A3 "Capacity\|Allocatable" | grep -i gpu

# Existing GPU pods (identify the 8GB "other project")
kubectl get pods -A
nvidia-smi --query-compute-apps=pid,name,used_memory --format=csv

# Disk space (need 25GB)
df -h / /var/lib/containerd /olares 2>/dev/null

# System memory
free -h
```

**Decision gate 1**:
- All green → proceed to Phase 1
- CUDA < 12.8 → STOP, user must upgrade driver first
- Olares < 1.12.6 → STOP, user must upgrade Olares
- Other project uses > 8GB VRAM → STOP, discuss with user

### Phase 1: Project directory + source code (10 min)

**On Olares One**:
```bash
sudo mkdir -p /opt/bdvoice-agent/{charts,images,models,scripts,logs}
sudo chown -R $USER:$USER /opt/bdvoice-agent
cd /opt/bdvoice-agent
```

**Get source code**:
Option A - if user has the zip accessible:
```bash
unzip ~/Downloads/bdvoice-agent-platform-master.zip -d /opt/bdvoice-agent/src
```

Option B - clone from user's git if public:
```bash
git clone https://github.com/rafiulislam4246/bdvoiceagent /opt/bdvoice-agent/src
# (user owns this repo per memory [[kotha-production-deploy]])
```

Verify source layout:
```bash
ls /opt/bdvoice-agent/src/bdvoice-agent-platform-master/
# Expected: ears.py, mouth.py, brain.py, agentcfg.py, bangla_text.py, etc.
```

### Phase 2: Model pre-download (20-30 min)

Models go to `/olares/common/bdvoice-models/` so all 3 pods can share without duplicating.

```bash
cat > /opt/bdvoice-agent/scripts/download-models.sh << 'EOF'
#!/bin/bash
set -e

MODELS_DIR="/olares/common/bdvoice-models"
sudo mkdir -p "$MODELS_DIR"
sudo chown -R $USER:$USER "$MODELS_DIR"
cd "$MODELS_DIR"

# Need git-lfs for HuggingFace model downloads
if ! command -v git-lfs &> /dev/null; then
    sudo apt-get install -y git-lfs
fi
git lfs install

echo "=== 1/3: Ear model (bn-regional-ct2, ~3GB) ==="
if [ ! -d "bn-regional-ct2/model.bin" ]; then
    git clone https://huggingface.co/IamSanjid/tugstugi_bengaliai-regional-asr_whisper-medium-ct2 bn-regional-ct2
fi

echo "=== 2/3: Mouth model (IndicF5_Bangladeshi, ~3GB) ==="
if [ ! -d "indicf5-bangladeshi/model_1250000.safetensors" ]; then
    git clone https://huggingface.co/actorcritic/IndicF5_Bangladeshi indicf5-bangladeshi
fi

echo "=== 3/3: Brain model (Qwen2.5-7B Q6 via Ollama, ~6GB) ==="
mkdir -p "$MODELS_DIR/ollama"
docker run --rm --gpus all \
    -v "$MODELS_DIR/ollama:/root/.ollama" \
    ollama/ollama:latest \
    pull qwen2.5:7b-instruct-q6_K

echo "=== Verification ==="
du -sh "$MODELS_DIR"/*/
echo "Expected approx: 3GB ear, 3GB mouth, 7GB ollama"
EOF

chmod +x /opt/bdvoice-agent/scripts/download-models.sh
/opt/bdvoice-agent/scripts/download-models.sh 2>&1 | tee /opt/bdvoice-agent/logs/download.log
```

**Expected output**:
- ~3GB bn-regional-ct2
- ~3GB indicf5-bangladeshi  
- ~7GB ollama models

**Decision gate 2**:
- All 3 downloaded successfully → proceed
- Any failure → investigate before continuing

### Phase 3: Build Docker images (30 min)

Three Dockerfiles. ALL use CUDA 12.8 base for Blackwell compatibility.

#### 3.1 Ear Dockerfile

```bash
mkdir -p /opt/bdvoice-agent/images/ear
cat > /opt/bdvoice-agent/images/ear/Dockerfile << 'EOF'
FROM nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3.11 python3-pip python3.11-venv \
    ffmpeg libsndfile1 curl \
    && rm -rf /var/lib/apt/lists/*

RUN ln -sf /usr/bin/python3.11 /usr/local/bin/python3 && \
    ln -sf /usr/bin/python3.11 /usr/local/bin/python

WORKDIR /app

RUN pip install --no-cache-dir \
    faster-whisper==1.0.3 \
    ctranslate2==4.4.0 \
    aiohttp==3.9.5 \
    numpy==1.26.4 \
    pyyaml==6.0.2 \
    huggingface-hub==0.24.6

COPY ears.py agentcfg.py /app/

ENV PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:False
ENV MODEL_PATH=/models/bn-regional-ct2
ENV EARS=bn-regional-ct2
ENV EARS_PORT=8300

EXPOSE 8300

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8300/health || exit 1

CMD ["python3", "ears.py"]
EOF

# Copy source files into build context
cp /opt/bdvoice-agent/src/bdvoice-agent-platform-master/ears.py /opt/bdvoice-agent/images/ear/
cp /opt/bdvoice-agent/src/bdvoice-agent-platform-master/agentcfg.py /opt/bdvoice-agent/images/ear/
```

#### 3.2 Mouth Dockerfile

```bash
mkdir -p /opt/bdvoice-agent/images/mouth
cat > /opt/bdvoice-agent/images/mouth/Dockerfile << 'EOF'
FROM nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3.11 python3-pip python3.11-venv \
    ffmpeg libsndfile1 git curl \
    && rm -rf /var/lib/apt/lists/*

RUN ln -sf /usr/bin/python3.11 /usr/local/bin/python3 && \
    ln -sf /usr/bin/python3.11 /usr/local/bin/python

WORKDIR /app

# PyTorch 2.6+ for Blackwell
RUN pip install --no-cache-dir \
    torch==2.6.0 torchaudio==2.6.0 \
    --index-url https://download.pytorch.org/whl/cu128

RUN pip install --no-cache-dir \
    aiohttp==3.9.5 \
    soundfile==0.12.1 \
    numpy==1.26.4 \
    pyyaml==6.0.2 \
    huggingface-hub==0.24.6

# F5-TTS from AI4Bharat fork (has IndicF5 support)
RUN pip install --no-cache-dir \
    git+https://github.com/AI4Bharat/f5_tts.git

COPY mouth.py bangla_text.py agentcfg.py /app/
COPY ref/ /app/ref/

ENV PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:False
ENV F5_REF=/app/ref/short_26.wav
ENV F5_NFE=12
ENV F5_SPEED=1.15
ENV MOUTH=indicf5
ENV MOUTH_PORT=8301

EXPOSE 8301

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8301/health || exit 1

CMD ["python3", "mouth.py"]
EOF

# Copy source files
cp /opt/bdvoice-agent/src/bdvoice-agent-platform-master/mouth.py /opt/bdvoice-agent/images/mouth/
cp /opt/bdvoice-agent/src/bdvoice-agent-platform-master/bangla_text.py /opt/bdvoice-agent/images/mouth/
cp /opt/bdvoice-agent/src/bdvoice-agent-platform-master/agentcfg.py /opt/bdvoice-agent/images/mouth/
cp -r /opt/bdvoice-agent/src/bdvoice-agent-platform-master/ref /opt/bdvoice-agent/images/mouth/
```

#### 3.3 Brain uses Ollama official image (no custom build)

#### Build + verify
```bash
cd /opt/bdvoice-agent/images
docker build -t bdvoice-ear:v1 ./ear 2>&1 | tee ../logs/build-ear.log
docker build -t bdvoice-mouth:v1 ./mouth 2>&1 | tee ../logs/build-mouth.log
docker pull ollama/ollama:latest

docker images | grep -E "bdvoice|ollama"
# Expect: bdvoice-ear:v1, bdvoice-mouth:v1, ollama/ollama:latest
```

**Decision gate 3**:
- Both custom images built without errors → proceed
- Build errors → check logs, usually CUDA version mismatch, fix Dockerfile FROM line

### Phase 4: Olares chart manifests (45 min)

Three separate apps. Full manifests in `~/Work/tts-stt-research/olares-deployment-plan.md` (reference it, don't retype from memory).

**Key per manifest**:
- `OlaresManifest.yaml` with `spec.accelerator` GPU claim
- `templates/deployment.yaml` with k8s Deployment + Service
- `permission.appCommon: true` for model folder access

**Follow olares-deployment-plan.md Section Phase 4 exactly** - it has all 3 chart YAML specs ready to copy.

**Package each chart**:
```bash
cd /opt/bdvoice-agent/charts
tar -czf bdvoice-brain.tgz bdvoice-brain/
tar -czf bdvoice-mouth.tgz bdvoice-mouth/
tar -czf bdvoice-ear.tgz bdvoice-ear/
```

### Phase 5: Deploy (STRICT ORDER, 30s gaps)

**CRITICAL**: Follow this order exactly. Time-slicing GPU sharing demands cold-start ordering.

```bash
# 1. BRAIN FIRST (biggest, loads 6GB model)
olares-chart install bdvoice-brain.tgz
echo "Waiting 30s for brain cold-start..."
sleep 30

# Verify brain healthy
kubectl get pods -l app=bdvoice-brain
kubectl logs -l app=bdvoice-brain --tail=20
# Look for: "Ollama server listening on 0.0.0.0:11434"

# Direct brain test
kubectl exec -it deployment/bdvoice-brain -- \
    curl -s http://localhost:11434/api/generate \
    -d '{"model":"qwen2.5:7b-instruct-q6_K","prompt":"নমস্কার","stream":false}' \
    | head -40
# Expect: JSON response within 2-3 sec with actual Bangla text

# 2. MOUTH SECOND
olares-chart install bdvoice-mouth.tgz
sleep 30
kubectl get pods -l app=bdvoice-mouth
kubectl logs -l app=bdvoice-mouth --tail=20

# 3. EAR LAST
olares-chart install bdvoice-ear.tgz
sleep 30
kubectl get pods -l app=bdvoice-ear
kubectl logs -l app=bdvoice-ear --tail=20

# Verify GPU usage
nvidia-smi
# Expected: ~11-12 GB used, ~12-13 GB free (8GB for other project + 4GB buffer)
```

**Decision gate 4**: After EACH service deploys, ensure:
- Pod status = Running
- GPU memory incremented as expected (+6GB brain, +3GB mouth, +2GB ear)
- No OOM errors in logs
- Other project's 8GB untouched

If any gate fails → `olares-chart uninstall <last-installed>` and debug before continuing.

### Phase 6: Smoke test (15 min)

```bash
cat > /opt/bdvoice-agent/scripts/smoke-test.sh << 'EOF'
#!/bin/bash
set -e

# Port-forward for local testing
kubectl port-forward svc/ear 8300:8300 &
PF_EAR=$!
kubectl port-forward svc/mouth 8301:8301 &
PF_MOUTH=$!
kubectl port-forward svc/brain 11434:11434 &
PF_BRAIN=$!
sleep 3

cleanup() {
    kill $PF_EAR $PF_MOUTH $PF_BRAIN 2>/dev/null || true
}
trap cleanup EXIT

EAR=http://localhost:8300
MOUTH=http://localhost:8301
BRAIN=http://localhost:11434

echo "=== Health checks ==="
curl -s $EAR/health | head -5
curl -s $MOUTH/health | head -5
curl -s $BRAIN/api/tags | head -10

echo ""
echo "=== Full voice turn test ==="

# 1. Mouth synthesizes question
echo "--- 1. Mouth synthesize ---"
T1=$(date +%s%N)
curl -s -X POST $MOUTH/say \
    -H 'content-type: application/json' \
    -d '{"text":"আমার ব্যালেন্স কত?"}' \
    --output /tmp/question.wav
T2=$(date +%s%N)
MOUTH_MS=$(( (T2 - T1) / 1000000 ))
echo "Mouth: ${MOUTH_MS}ms"
ls -la /tmp/question.wav

# 2. Ear transcribes
echo "--- 2. Ear transcribe ---"
T1=$(date +%s%N)
RESULT=$(curl -s -F "audio=@/tmp/question.wav" $EAR/listen)
T2=$(date +%s%N)
EAR_MS=$(( (T2 - T1) / 1000000 ))
echo "Ear: ${EAR_MS}ms"
echo "Transcript: $RESULT"

# 3. Brain generates response
echo "--- 3. Brain response ---"
T1=$(date +%s%N)
BRAIN_RESP=$(curl -s $BRAIN/api/generate \
    -d '{"model":"qwen2.5:7b-instruct-q6_K","prompt":"User: আমার ব্যালেন্স কত?\nBot:","stream":false}')
T2=$(date +%s%N)
BRAIN_MS=$(( (T2 - T1) / 1000000 ))
echo "Brain: ${BRAIN_MS}ms"
echo "Response: $(echo $BRAIN_RESP | head -200)"

TOTAL=$(( EAR_MS + BRAIN_MS + MOUTH_MS ))
echo ""
echo "============================="
echo "TOTAL: ${TOTAL}ms"
echo "Target: under 1500ms"
if [ $TOTAL -lt 1500 ]; then
    echo "PASS"
else
    echo "OVER TARGET - investigate"
fi
EOF

chmod +x /opt/bdvoice-agent/scripts/smoke-test.sh
/opt/bdvoice-agent/scripts/smoke-test.sh 2>&1 | tee /opt/bdvoice-agent/logs/smoke.log
```

**Expected result**:
- Mouth: 350-450ms
- Ear: 150-250ms
- Brain: 150-250ms
- **TOTAL: 650-950ms**

**Decision gate 5**:
- TOTAL < 1500ms → PASS, proceed to Phase 7
- 1500-2000ms → investigate which stage is slow, don't ship
- > 2000ms → rollback, something structural wrong

### Phase 7: Production verification (30 min)

#### 7.1 Other project verification
Ask user to test their other 8GB project. Must confirm:
- Still works as before
- No OOM errors
- Performance unchanged

#### 7.2 Concurrent load test
```bash
# 2 parallel voice turns
./smoke-test.sh &
./smoke-test.sh &
wait
# Both should succeed under 1500ms each
```

#### 7.3 24-hour soak test
```bash
cat > /opt/bdvoice-agent/scripts/soak-test.sh << 'EOF'
for i in $(seq 1 1440); do
    /opt/bdvoice-agent/scripts/smoke-test.sh > /opt/bdvoice-agent/logs/soak-$(date +%Y%m%d-%H%M).log 2>&1
    sleep 60
done
EOF
chmod +x /opt/bdvoice-agent/scripts/soak-test.sh
nohup /opt/bdvoice-agent/scripts/soak-test.sh > /opt/bdvoice-agent/logs/soak.log 2>&1 &
echo $! > /opt/bdvoice-agent/logs/soak.pid
```

User monitors over 24 hours. Success criteria:
- 95% of turns < 1500ms
- No OOM crashes
- Other project still works
- GPU usage steady, no drift

### Phase 8: Rollback (ONLY if things go wrong)

```bash
# Safe rollback, 5 min
olares-chart uninstall bdvoice-ear
olares-chart uninstall bdvoice-mouth
olares-chart uninstall bdvoice-brain

# Verify VRAM returned
nvidia-smi

# Models stay in /olares/common/bdvoice-models (don't delete)
# Images stay in docker (don't delete)
# Can re-deploy quickly after fixing issue
```

---

## 4. ERROR HANDLING GUIDANCE

### Common errors you'll hit

#### Build error: "CUDA version not supported"
- Cause: using older nvidia/cuda image
- Fix: ensure `nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04` in FROM line
- Rebuild: `docker build --no-cache -t bdvoice-ear:v1 ./ear`

#### Pod OOM (OOMKilled status)
- Cause: GPU memory exceeded 16GB budget
- Immediate: uninstall last app, reduce `limitedGPUMemory`
- Investigate: `kubectl describe pod` to see memory at crash
- Fix options:
  - Reduce brain quant: Q6 → Q4 (-2GB)
  - Switch to smaller brain: 7B → 3B (titulm)
  - Reduce F5 NFE: 12 → 10 (slight quality loss)

#### Pod stuck in ContainerCreating
- Cause: image pull failed or volume mount issue
- Check: `kubectl describe pod <name>`
- Common fix: ensure `/olares/common/bdvoice-models` exists and has correct ownership

#### "GPU not found in pod"
- Cause: nvidia-container-toolkit not configured on node
- Check: `kubectl get nodes -o yaml | grep -A5 accelerator`
- Fix: `olares-cli gpu install && olares-cli gpu enable`

#### Brain responds with garbage/empty
- Cause: wrong model name in Ollama
- Check: `kubectl exec deployment/bdvoice-brain -- ollama list`
- Fix: re-pull with exact tag `qwen2.5:7b-instruct-q6_K`

#### High latency (brain > 500ms)
- Cause: cold GPU state or P-state downclock
- Fix: `nvidia-smi -pm 1 && sudo nvidia-smi -lgc <max_clock>`
- Check: `nvidia-smi --query-gpu=pstate --format=csv`
- Expected: P0/P2, not P8

### What to do when stuck

1. **STOP and report to user** - don't try risky fixes alone
2. Share error output verbatim  
3. Suggest 2-3 options, let user pick
4. Never run destructive commands without approval

---

## 5. FALLBACK PLAN (if Olares deploy fundamentally breaks)

### Option A: Docker Compose on same hardware
If Olares k3s is broken, bypass Olares layer:

```bash
cat > /opt/bdvoice-agent/docker-compose.yml << 'EOF'
version: '3.8'
services:
  brain:
    image: ollama/ollama:latest
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - OLLAMA_MODELS=/root/.ollama
    ports: ["11434:11434"]
    volumes: ["/olares/common/bdvoice-models/ollama:/root/.ollama"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  mouth:
    image: bdvoice-mouth:v1
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - F5_CKPT=/models/indicf5-bangladeshi
    ports: ["8301:8301"]
    volumes: ["/olares/common/bdvoice-models:/models"]

  ear:
    image: bdvoice-ear:v1
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - MODEL_PATH=/models/bn-regional-ct2
    ports: ["8300:8300"]
    volumes: ["/olares/common/bdvoice-models:/models"]
EOF

docker compose -f /opt/bdvoice-agent/docker-compose.yml up -d
```

Loses Olares UI/Market features. Gains:
- Docker Compose battle-tested
- Direct nvidia runtime
- Easier debug
- Can migrate back to Olares charts later

### Option B: Native Python on host (if Docker also breaks)
```bash
# Install Python 3.11 + CUDA 12.8 PyTorch on bare Olares host
# Run ears.py, mouth.py as systemd services
# Ollama via official installer
# No containers, maximum compatibility
```

---

## 6. SUCCESS CRITERIA

### At end of deploy, prove:
- [ ] `kubectl get pods` shows 3 Running pods (brain, mouth, ear)
- [ ] `nvidia-smi` shows 11-13 GB used, 11-13 GB free
- [ ] Smoke test total < 1500ms consistently
- [ ] Other project's 8GB untouched, still working
- [ ] User performed 1 real voice turn and confirmed quality
- [ ] 24h soak test started

### What "done" looks like
Rafiul says: "Kaj kore amader standard er sathe match kore" (works and matches our standards). NOT: "HTTP 200 returned."

---

## 7. REPORTING TO USER

### During deploy, after each phase:
```
Phase <N> done. 
Checkpoint: <what we verified>
Result: <PASS/FAIL>
Next: <what happens next>
Need approval? <YES/NO>
```

### If something breaks:
```
ERROR in Phase <N>.
What failed: <command/step>
Error output: <paste verbatim>
My guess at cause: <hypothesis>
Suggested fix: <option A or B or STOP>
Waiting for your call.
```

### At end:
```
Deploy done. 
Pods: brain, mouth, ear all Running
Latency: <ear>/<brain>/<mouth> = <total>ms
GPU: <used>/<total> GB
Other project: <status>
Soak test: started, monitor at <path>
Known caveats: <list>
```

---

## 8. REFERENCE LINKS

### Documentation (public)
- Olares docs: https://www.olares.com/docs/
- Olares manifest spec: https://www.olares.com/docs/developer/develop/package/manifest
- Olares GPU config: https://www.olares.com/docs/one/configure-gpu-mode
- NVIDIA container toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/
- F5-TTS repo: https://github.com/SWivid/F5-TTS
- faster-whisper: https://github.com/SYSTRAN/faster-whisper
- Ollama: https://ollama.com/docs

### Research notes (user has these)
- Full gap analysis: `/Users/rafiulislam/Work/tts-stt-research/verbex-vs-our-stack-full-gap.md`
- Olares deployment plan (predecessor): `/Users/rafiulislam/Work/tts-stt-research/olares-deployment-plan.md`
- Verbex architecture: `/Users/rafiulislam/Work/tts-stt-research/verbex-voice-agent-architecture.md`
- All research notes GitHub: https://github.com/rafiulislam4246/bd-bangla-voice-research

### User's own code
- Current voice agent: `~/Downloads/bdvoice-agent-platform-master.zip`
- Kotha platform: https://github.com/rafiulislam4246/bdvoiceagent (public)
- Memory system: `/Users/rafiulislam/.claude/projects/-Users-rafiulislam-Work-n8n/memory/`

### Key memory entries relevant to this deploy
- `[[verbex-architecture-reference]]` - what we're replacing
- `[[kotha-production-deploy]]` - Kotha live state
- `[[ecomsixx-bangla-voice-plan]]` - voice roadmap
- `[[feedback_no_push_without_permission]]` - deploy rule
- `[[feedback_verify_before_declaring_done]]` - verification rule

---

## 9. QUICK REFERENCE CARD (for user)

```bash
# Status check
kubectl get pods -l 'app in (bdvoice-brain,bdvoice-mouth,bdvoice-ear)'
nvidia-smi

# View logs (last 50 lines)
kubectl logs -l app=bdvoice-brain --tail=50
kubectl logs -l app=bdvoice-mouth --tail=50
kubectl logs -l app=bdvoice-ear --tail=50

# Follow logs live
kubectl logs -l app=bdvoice-brain -f

# Restart one service
kubectl rollout restart deployment/bdvoice-brain

# Full uninstall
olares-chart uninstall bdvoice-ear
olares-chart uninstall bdvoice-mouth
olares-chart uninstall bdvoice-brain

# Re-run smoke test
/opt/bdvoice-agent/scripts/smoke-test.sh

# Check soak test progress
tail -f /opt/bdvoice-agent/logs/soak.log
ls /opt/bdvoice-agent/logs/soak-*.log | wc -l  # how many runs
```

---

## 10. FINAL NOTES TO FUTURE CLAUDE

1. **Read olares-deployment-plan.md FIRST** - it has full chart YAML specs that are too long to duplicate here
2. **User knows Olares + has access** - don't re-explain what Olares is
3. **Reply in Banglish** - user prefers this, feels less distant
4. **No em-dashes** - user hates them, grep before sending output
5. **Verify every claim** - user got burned by hallucinated optimization advice in past
6. **8GB reserved VRAM is sacred** - if our pods touch it, user loses trust + other project breaks
7. **NFE=12 for F5-TTS stays** - don't try to drop to 8 (I did, was wrong, see `latency-optimization.md`)
8. **Brain = Qwen2.5-7B Q6** - decided after much research, don't suggest alternatives mid-deploy
9. **Olares uses time-slicing GPU** - NO memory isolation, be strict with limits
10. **24h soak test before calling it production-ready** - user's culture, respect it

---

## 11. CONVERSATION STARTER

When user opens Claude on the other laptop and starts a session, open with:

> Rafiul, Olares One e Bangla voice agent deploy korar jonno ami ekta step-by-step plan ready korechi. Prothome environment check korte hobe - oi commands run koro ar output pathao. Full plan at [path to this doc]. 
>
> Start with Phase 0 - 7 ta command. Taar output dekhlei age baar jabo.

Then wait for Phase 0 output. Follow the phases in order. Report after each. Ask approval before each gate. Don't skip verifications.

---

**END OF GUIDE**

Total pages in this guide: ~15
Total deploy time: ~3 hours active work + 24h soak
Expected outcome: 650-950ms first-sound Bangla voice agent, zero touch to other project's 8GB.

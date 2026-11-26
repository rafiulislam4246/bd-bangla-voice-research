# Olares One Deployment Plan - Bangla Voice Agent Stack

**Target hardware**: Olares One (RTX 5090 Mobile 24GB, Intel Ultra 9 275HX, 96GB RAM, 2TB NVMe)
**Date prepared**: 2026-11-26
**Deploy window**: Tonight (per plan)
**Risk level**: YELLOW (Olares + Blackwell, proceed with gates)

---

## Final architecture decision

### Stack:
- **Ear (STT)**: bn-regional-ct2 via faster-whisper + CTranslate2 (~2GB VRAM)
- **Mouth (TTS)**: IndicF5 via F5-TTS + Vocos (~3GB VRAM)  
- **Brain (LLM)**: **Qwen2.5-7B Instruct Q6** via Ollama (~6GB VRAM)

### VRAM budget (24GB total):
```
Ear:            2GB
Mouth:          3GB
Brain (7B Q6):  6GB
System buffer:  1GB
─────────────────
Our usage:     12GB
Reserved for other project:  8GB
Safety margin:  4GB  ✅ comfortable
```

### Projected latency (based on RTX 5090 Mobile ~3x faster than 4070 Laptop):
```
Ear:   ~150-200ms
Brain: ~150-250ms  
Mouth: ~350-450ms
───────────────────
First-sound: ~650-900ms  ← well under 1,500ms target
```

Beats Verbex (2,235-2,515ms) by 1,300-1,600ms.

---

## Phase 0: Pre-flight environment check (10 min)

Run on Olares One terminal:

```bash
# 1. Verify GPU + driver
nvidia-smi
# Must see: "NVIDIA GeForce RTX 5090 Mobile", 24576 MiB, driver ≥570

# 2. CUDA version
nvidia-smi | grep "CUDA Version"
# Must be: 12.8 or higher (Blackwell requirement)

# 3. Olares version  
olares-cli version
# Must be: 1.12.6+

# 4. Identify the 8GB-using "other project"
nvidia-smi --query-compute-apps=pid,name,used_memory --format=csv
# Note what it is - we schedule around it

# 5. k3s GPU scheduling
kubectl get nodes -o yaml | grep -A5 accelerator
kubectl get pods -A | grep -i "gpu\|nvidia"

# 6. Disk space (need ~25GB for models + containers)
df -h / /var/lib/containerd 2>/dev/null || df -h /

# 7. System memory (we have 96GB, plenty)
free -h
```

### Go/No-Go gate:
- ✅ All checks pass → proceed to Phase 1
- ❌ CUDA < 12.8 → upgrade driver first (don't proceed)
- ❌ Olares < 1.12.6 → upgrade Olares first (don't proceed)  
- ❌ Other project uses > 8GB consistently → reduce brain to 3B model

---

## Phase 1: Project directory setup (5 min)

```bash
# Create working directory on Olares One
mkdir -p /opt/bdvoice-agent/{charts,images,models,scripts,logs}
cd /opt/bdvoice-agent

# This becomes our deployment workspace
```

---

## Phase 2: Model pre-download (one-time, ~20 min)

Models downloaded to `appCommon` folder so pods can share them without duplicating.

### Create model download script

```bash
cat > /opt/bdvoice-agent/scripts/download-models.sh << 'EOF'
#!/bin/bash
set -e

MODELS_DIR="/olares/common/bdvoice-models"
mkdir -p "$MODELS_DIR"
cd "$MODELS_DIR"

echo "=== Downloading Ear model (bn-regional-ct2) ==="
if [ ! -d "bn-regional-ct2" ]; then
    git lfs install
    git clone https://huggingface.co/IamSanjid/tugstugi_bengaliai-regional-asr_whisper-medium-ct2 bn-regional-ct2
fi

echo "=== Downloading Mouth model (IndicF5_Bangladeshi) ==="
if [ ! -d "indicf5-bangladeshi" ]; then
    git lfs install
    git clone https://huggingface.co/actorcritic/IndicF5_Bangladeshi indicf5-bangladeshi
fi

echo "=== Downloading Brain model (Qwen2.5-7B Q6) via Ollama ==="
# Will be pulled by Ollama container on first start
# Pre-pull to appCommon:
docker run --rm -v "$MODELS_DIR/ollama:/root/.ollama" \
    --gpus all \
    ollama/ollama:latest \
    pull qwen2.5:7b-instruct-q6_K

echo "=== Verify sizes ==="
du -sh "$MODELS_DIR"/*/
echo "Expected: ~3GB ear, ~3GB mouth, ~7GB ollama"
EOF

chmod +x /opt/bdvoice-agent/scripts/download-models.sh
```

### Run download (one time):
```bash
/opt/bdvoice-agent/scripts/download-models.sh
# Takes ~20 min on good connection
```

---

## Phase 3: Build Docker images (30 min)

Three images, all CUDA 12.8 base for Blackwell compatibility.

### 3.1 Ear image (`bdvoice-ear`)

```dockerfile
# /opt/bdvoice-agent/images/ear/Dockerfile
FROM nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3.11 python3-pip python3.11-venv \
    ffmpeg libsndfile1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install faster-whisper + CTranslate2 with CUDA 12 support
RUN pip install --no-cache-dir \
    faster-whisper==1.0.3 \
    ctranslate2==4.4.0 \
    aiohttp==3.9.5 \
    numpy==1.26.4

# Copy ear service code
COPY ears.py agentcfg.py /app/

ENV PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:False
ENV MODEL_PATH=/models/bn-regional-ct2
ENV EARS_PORT=8300

EXPOSE 8300

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8300/health || exit 1

CMD ["python3", "ears.py"]
```

### 3.2 Mouth image (`bdvoice-mouth`)

```dockerfile
# /opt/bdvoice-agent/images/mouth/Dockerfile
FROM nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3.11 python3-pip python3.11-venv \
    ffmpeg libsndfile1 git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# PyTorch 2.6+ for Blackwell
RUN pip install --no-cache-dir \
    torch==2.6.0 torchaudio==2.6.0 \
    --index-url https://download.pytorch.org/whl/cu128

# F5-TTS from AI4Bharat fork (has IndicF5 support)
RUN pip install --no-cache-dir \
    git+https://github.com/AI4Bharat/f5_tts.git \
    aiohttp==3.9.5 \
    soundfile==0.12.1

COPY mouth.py bangla_text.py agentcfg.py /app/
COPY ref/short_26.wav /app/ref/

ENV PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:False
ENV F5_REF=/app/ref/short_26.wav
ENV F5_NFE=12
ENV F5_SPEED=1.15
ENV MOUTH_PORT=8301

EXPOSE 8301

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8301/health || exit 1

CMD ["python3", "mouth.py"]
```

### 3.3 Brain = Ollama official image (no custom build)

Use `ollama/ollama:latest` directly - validated path per Olares Market. Brain configuration via env vars.

### Build commands:
```bash
cd /opt/bdvoice-agent/images

docker build -t bdvoice-ear:v1 ./ear
docker build -t bdvoice-mouth:v1 ./mouth
docker pull ollama/ollama:latest

# Verify images
docker images | grep -E "bdvoice|ollama"
```

---

## Phase 4: Olares chart manifests (45 min)

Three separate Olares apps. Each with own GPU resource claim.

### 4.1 Brain chart (`bdvoice-brain`)

```yaml
# /opt/bdvoice-agent/charts/bdvoice-brain/OlaresManifest.yaml
olaresManifest.version: '0.8.0'
olaresManifest.type: app

metadata:
  name: bdvoice-brain
  description: 'Bangla LLM brain (Qwen2.5-7B Q6)'
  icon: https://example.com/brain.png
  appid: bdvoice-brain
  version: '1.0.0'
  categories:
    - AI

entrances:
  - name: brain-api
    port: 11434
    host: brain
    title: Brain API
    authLevel: internal   # internal only, other pods can reach it

permission:
  appCommon: true   # share model folder across apps

spec:
  versionName: '1.0.0'
  featuredImage: https://example.com/featured.png
  accelerator:
  - mode: nvidia
    requiredGPUMemory: 6Gi
    limitedGPUMemory: 7Gi    # hard cap near our 6GB target
    requiredMemory: 4Gi
    limitedMemory: 8Gi
    requiredCpu: 500m
    limitedCpu: 4000m
```

```yaml
# /opt/bdvoice-agent/charts/bdvoice-brain/templates/deployment.yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bdvoice-brain
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bdvoice-brain
  template:
    metadata:
      labels:
        app: bdvoice-brain
    spec:
      containers:
      - name: brain
        image: ollama/ollama:latest
        env:
        - name: OLLAMA_MODELS
          value: "/models/ollama"
        - name: OLLAMA_MAX_LOADED_MODELS  
          value: "1"
        - name: OLLAMA_NUM_PARALLEL
          value: "2"
        - name: OLLAMA_KEEP_ALIVE
          value: "10m"
        - name: PYTORCH_CUDA_ALLOC_CONF
          value: "max_split_size_mb:512,expandable_segments:False"
        volumeMounts:
        - name: models
          mountPath: /models
        ports:
        - containerPort: 11434
        resources:
          requests:
            nvidia.com/gpu: 1
      volumes:
      - name: models
        hostPath:
          path: /olares/common/bdvoice-models
---
apiVersion: v1
kind: Service
metadata:
  name: brain
spec:
  selector:
    app: bdvoice-brain
  ports:
  - port: 11434
    targetPort: 11434
```

### 4.2 Mouth chart (`bdvoice-mouth`)

```yaml
# /opt/bdvoice-agent/charts/bdvoice-mouth/OlaresManifest.yaml
olaresManifest.version: '0.8.0'
olaresManifest.type: app

metadata:
  name: bdvoice-mouth
  description: 'Bangla TTS (IndicF5)'
  appid: bdvoice-mouth
  version: '1.0.0'
  categories: [AI]

entrances:
  - name: mouth-api
    port: 8301
    host: mouth
    title: Mouth API
    authLevel: internal

permission:
  appCommon: true

spec:
  accelerator:
  - mode: nvidia
    requiredGPUMemory: 3Gi
    limitedGPUMemory: 4Gi
    requiredMemory: 3Gi
    limitedMemory: 6Gi
    requiredCpu: 300m
    limitedCpu: 2000m
```

```yaml
# /opt/bdvoice-agent/charts/bdvoice-mouth/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bdvoice-mouth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bdvoice-mouth
  template:
    metadata:
      labels:
        app: bdvoice-mouth
    spec:
      containers:
      - name: mouth
        image: bdvoice-mouth:v1
        env:
        - name: F5_CKPT
          value: "/models/indicf5-bangladeshi"
        - name: F5_REF
          value: "/app/ref/short_26.wav"
        - name: F5_NFE
          value: "12"
        volumeMounts:
        - name: models
          mountPath: /models
        ports:
        - containerPort: 8301
        resources:
          requests:
            nvidia.com/gpu: 1
      volumes:
      - name: models
        hostPath:
          path: /olares/common/bdvoice-models
---
apiVersion: v1
kind: Service
metadata:
  name: mouth
spec:
  selector:
    app: bdvoice-mouth
  ports:
  - port: 8301
    targetPort: 8301
```

### 4.3 Ear chart (`bdvoice-ear`)

```yaml
# /opt/bdvoice-agent/charts/bdvoice-ear/OlaresManifest.yaml
olaresManifest.version: '0.8.0'
olaresManifest.type: app

metadata:
  name: bdvoice-ear
  description: 'Bangla STT (bn-regional-ct2)'
  appid: bdvoice-ear
  version: '1.0.0'
  categories: [AI]

entrances:
  - name: ear-api
    port: 8300
    host: ear
    title: Ear API
    authLevel: internal

permission:
  appCommon: true

spec:
  accelerator:
  - mode: nvidia
    requiredGPUMemory: 2Gi
    limitedGPUMemory: 3Gi
    requiredMemory: 2Gi
    limitedMemory: 4Gi
    requiredCpu: 200m
    limitedCpu: 1500m
```

```yaml
# /opt/bdvoice-agent/charts/bdvoice-ear/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bdvoice-ear
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bdvoice-ear
  template:
    metadata:
      labels:
        app: bdvoice-ear
    spec:
      containers:
      - name: ear
        image: bdvoice-ear:v1
        env:
        - name: MODEL_PATH
          value: "/models/bn-regional-ct2"
        volumeMounts:
        - name: models
          mountPath: /models
        ports:
        - containerPort: 8300
        resources:
          requests:
            nvidia.com/gpu: 1
      volumes:
      - name: models
        hostPath:
          path: /olares/common/bdvoice-models
---
apiVersion: v1
kind: Service
metadata:
  name: ear
spec:
  selector:
    app: bdvoice-ear
  ports:
  - port: 8300
    targetPort: 8300
```

### Package charts:
```bash
cd /opt/bdvoice-agent/charts
tar -czf bdvoice-brain.tgz bdvoice-brain/
tar -czf bdvoice-mouth.tgz bdvoice-mouth/
tar -czf bdvoice-ear.tgz bdvoice-ear/
```

---

## Phase 5: Deployment (sequence matters)

**CRITICAL: deploy IN THIS ORDER with 30s gaps** (per research: cold-start ordering matters under time-slicing)

```bash
# 1. Brain FIRST (biggest GPU consumer, loads model)
olares-chart install bdvoice-brain.tgz
echo "Waiting 30s for brain to stabilize..."
sleep 30

# Verify brain is ready
kubectl get pods -l app=bdvoice-brain
kubectl logs -l app=bdvoice-brain --tail=20
# Expect to see "Ollama server listening on 0.0.0.0:11434"

# Test brain
kubectl exec -it deployment/bdvoice-brain -- \
    curl -s http://localhost:11434/api/generate \
    -d '{"model":"qwen2.5:7b-instruct-q6_K","prompt":"হ্যালো","stream":false}'
# Expect JSON response within 2-3 sec

# 2. Mouth second
olares-chart install bdvoice-mouth.tgz
sleep 30
kubectl get pods -l app=bdvoice-mouth

# Test mouth
kubectl port-forward svc/mouth 8301:8301 &
curl http://localhost:8301/health

# 3. Ear last
olares-chart install bdvoice-ear.tgz
sleep 30
kubectl get pods -l app=bdvoice-ear

# Test ear
kubectl port-forward svc/ear 8300:8300 &
curl http://localhost:8300/health

# 4. Verify GPU usage
nvidia-smi
# Expect: ~11-12 GB used by our 3 pods, 8+ GB free for other project
```

---

## Phase 6: Smoke test (15 min)

Create end-to-end test script:

```bash
cat > /opt/bdvoice-agent/scripts/smoke-test.sh << 'EOF'
#!/bin/bash
# Full voice turn: synthesize question → transcribe → brain → synthesize answer

set -e

EAR=http://localhost:8300
MOUTH=http://localhost:8301
BRAIN=http://localhost:11434

echo "=== 1. Mouth health ==="
curl -s $MOUTH/health | jq .

echo "=== 2. Ear health ==="
curl -s $EAR/health | jq .

echo "=== 3. Brain health ==="
curl -s $BRAIN/api/tags | jq .

echo "=== 4. Synthesize question ==="
TIME1=$(date +%s%N)
curl -s -X POST $MOUTH/say \
    -H 'content-type: application/json' \
    -d '{"text":"আমার ব্যালেন্স কত?"}' \
    --output /tmp/question.wav
TIME2=$(date +%s%N)
MOUTH_MS=$(( (TIME2 - TIME1) / 1000000 ))
echo "Mouth latency: ${MOUTH_MS}ms"

echo "=== 5. Transcribe ==="
TIME1=$(date +%s%N)
curl -s -F "audio=@/tmp/question.wav" $EAR/listen > /tmp/transcript.json
TIME2=$(date +%s%N)
EAR_MS=$(( (TIME2 - TIME1) / 1000000 ))
echo "Ear latency: ${EAR_MS}ms"
cat /tmp/transcript.json

echo "=== 6. Brain response ==="
TIME1=$(date +%s%N)
curl -s $BRAIN/api/generate \
    -d '{"model":"qwen2.5:7b-instruct-q6_K","prompt":"User: আমার ব্যালেন্স কত?\nBot:","stream":false}' \
    > /tmp/brain-response.json
TIME2=$(date +%s%N)
BRAIN_MS=$(( (TIME2 - TIME1) / 1000000 ))
echo "Brain latency: ${BRAIN_MS}ms"
cat /tmp/brain-response.json | jq .response

TOTAL=$(( EAR_MS + BRAIN_MS + MOUTH_MS ))
echo "=== TOTAL: ${TOTAL}ms ==="
echo "Target: under 1500ms"

if [ $TOTAL -lt 1500 ]; then
    echo "✅ PASS"
else
    echo "⚠️  Over target - investigate which stage is slow"
fi
EOF

chmod +x /opt/bdvoice-agent/scripts/smoke-test.sh
/opt/bdvoice-agent/scripts/smoke-test.sh
```

Expected output:
```
Mouth latency: 350-450ms
Ear latency: 150-250ms  
Brain latency: 150-250ms
TOTAL: 650-950ms
✅ PASS
```

---

## Phase 7: Production-readiness checks (30 min)

### 1. GPU memory verification
```bash
# Run while system idle
nvidia-smi --query-gpu=memory.used,memory.free --format=csv

# Should see:
# Used: ~12GB (our 3 pods + system)
# Free: ~12GB (8GB for other project + 4GB buffer)
```

### 2. Concurrent load test (2 parallel turns)
```bash
# Test that max_concurrency: 2 works
(curl -s -X POST $BRAIN/api/generate -d '{...}' &)
(curl -s -X POST $BRAIN/api/generate -d '{...}' &)
wait
# Both should complete without errors
```

### 3. 24-hour soak test
```bash
# Run 1 voice turn every 60 seconds for 24 hours
cat > /opt/bdvoice-agent/scripts/soak-test.sh << 'EOF'
for i in {1..1440}; do  # 1440 minutes = 24h
    /opt/bdvoice-agent/scripts/smoke-test.sh | tee -a /opt/bdvoice-agent/logs/soak-$i.log
    sleep 60
done
EOF

chmod +x /opt/bdvoice-agent/scripts/soak-test.sh
nohup /opt/bdvoice-agent/scripts/soak-test.sh > /opt/bdvoice-agent/logs/soak.log 2>&1 &
```

### 4. Other project verification
```bash
# Confirm the other 8GB project still works
# (Need user to run their own verification)
```

---

## Phase 8: Rollback (if anything goes wrong)

```bash
# Quick rollback (5 min)
olares-chart uninstall bdvoice-ear
olares-chart uninstall bdvoice-mouth
olares-chart uninstall bdvoice-brain

# Verify GPU memory returned
nvidia-smi

# System state: as if we never deployed
# Models remain in /olares/common/bdvoice-models (not deleted)
```

---

## Known risks + mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Brain OOM on long conversation | Low | Pod restart | `OLLAMA_KEEP_ALIVE=10m` + env var limits |
| Other project crashes | Medium | Both down | Strict `limitedGPUMemory`, monitor |
| Model download fails | Low | Deploy blocked | Retry script, pre-download before deploy |
| CUDA mismatch | Low | Container crash | CUDA 12.8 base images pinned |
| Pod networking issue | Low | Services can't reach each other | Use `authLevel: internal` entrances |
| Olares upgrade mid-deploy | Low | Breaks state | Don't upgrade during deploy window |

---

## Timeline estimate

| Phase | Time | Can parallelize? |
|---|---|---|
| 0. Env check | 10 min | No |
| 1. Directory setup | 5 min | No |
| 2. Model download | 20 min | Yes (run while reading Phase 3) |
| 3. Build Docker images | 30 min | Yes (3 can build parallel) |
| 4. Chart manifests | 45 min | No - sequential writing |
| 5. Deployment | 15 min | No - strict order |
| 6. Smoke test | 15 min | No |
| 7. Production checks | 30 min | Yes |
| 8. (Rollback if needed) | 5 min | No |

**Total active time**: ~2.5 hours
**Total elapsed**: ~3 hours (including download waits)

---

## Decision gates during deploy

### Gate 1: After Phase 0
- All checks pass → continue
- CUDA < 12.8 → upgrade driver first
- Other project > 8GB → use 3B brain instead

### Gate 2: After Phase 5 brain
- Brain responds in <500ms → continue to mouth
- Brain responds in >1000ms → investigate before mouth
- Brain OOM → rollback, reduce to smaller quant

### Gate 3: After Phase 6 smoke test  
- Total < 1500ms → PASS, deploy done
- Total 1500-2000ms → investigate, don't ship
- Total > 2000ms → rollback, debug

---

## Commands quick reference (copy-paste ready)

```bash
# Check status
kubectl get pods -l 'app in (bdvoice-brain,bdvoice-mouth,bdvoice-ear)'
nvidia-smi

# View logs  
kubectl logs -l app=bdvoice-brain --tail=50 -f
kubectl logs -l app=bdvoice-mouth --tail=50 -f
kubectl logs -l app=bdvoice-ear --tail=50 -f

# Restart a service
kubectl rollout restart deployment/bdvoice-brain

# Scale down (for debugging)
kubectl scale deployment/bdvoice-brain --replicas=0

# Full uninstall
olares-chart uninstall bdvoice-ear bdvoice-mouth bdvoice-brain
```

---

## Open items I'll need from you during deploy

1. **Olares admin credentials** - to install charts  
2. **Confirmation that 8GB other project is idle during install** (avoid conflict)
3. **Share any error messages immediately** - don't try to fix yourself, send to me
4. **Approval before each gate** - I'll ask "proceed?" at each checkpoint

---

## Expected end state

### Running pods:
- `bdvoice-brain` (Qwen2.5-7B Q6, 6GB VRAM, Ollama)
- `bdvoice-mouth` (IndicF5, 3GB VRAM)
- `bdvoice-ear` (bn-regional-ct2, 2GB VRAM)

### Accessible endpoints (from other pods):
- http://brain:11434 (Ollama API)
- http://mouth:8301 (TTS API)  
- http://ear:8300 (STT API)

### Metrics:
- Total GPU: 11-12GB used, 12-13GB free
- First-sound latency: 650-950ms per turn
- Other project: untouched 8GB, working

### Next steps after successful deploy:
1. Point your Kotha platform API at these new endpoints
2. Run 24h soak test
3. If stable: migrate first demo client from current setup to Olares stack
4. Consider: sell this as managed service to other BD AI companies (potential side business)

---

## Backup plan if Olares deploy fails

If after 3 hours we can't get Olares working:

### Fallback A: Docker Compose on same hardware (keeps Olares installed, bypasses)
```bash
cat > /opt/bdvoice-agent/docker-compose.yml << 'EOF'
version: '3.8'
services:
  brain:
    image: ollama/ollama:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports: ["11434:11434"]
    volumes: ["./models/ollama:/root/.ollama"]
    
  mouth:
    image: bdvoice-mouth:v1
    runtime: nvidia
    ports: ["8301:8301"]
    volumes: ["./models:/models"]
    
  ear:
    image: bdvoice-ear:v1  
    runtime: nvidia
    ports: ["8300:8300"]
    volumes: ["./models:/models"]
EOF

docker-compose up -d
```

Loses Olares UI/Market features but gains:
- Known-working Docker Compose
- NVIDIA runtime direct access
- Easier debugging
- Can migrate to Olares charts later

### Fallback B: Native Python on host
```bash
# Install Python 3.11 with CUDA 12.8 PyTorch
# Run ears.py, mouth.py, brain via Ollama systemd service
# No containers, maximum compatibility
```

---

## Checklist (print this)

- [ ] Phase 0: Environment check passed
- [ ] Phase 1: Directory created
- [ ] Phase 2: All 3 models downloaded to appCommon
- [ ] Phase 3: Docker images built
- [ ] Phase 4: Chart manifests written + packaged
- [ ] Phase 5: Brain installed and healthy
- [ ] Phase 5: Mouth installed and healthy
- [ ] Phase 5: Ear installed and healthy
- [ ] Phase 6: Smoke test passed (<1500ms)
- [ ] Phase 7: 24h soak test started
- [ ] Phase 7: Other project confirmed untouched
- [ ] Documentation: Kotha API updated to new endpoints

---

**Ready to execute tonight. Start with Phase 0 environment check. Share output, I'll confirm or adjust.**

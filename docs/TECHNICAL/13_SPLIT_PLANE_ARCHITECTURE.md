# Split-Plane Architecture - Hybrid Infrastructure

## Problema: "Compute Bound"

**Desafio:** Renderizar vídeo é caro. VPS de $35 não aguenta 10 canais renderizando simultaneamente sem travar o banco de dados e a API.

## Solução: Arquitetura Híbrida "Split-Plane"

Separamos o **Control Plane** (Leve, Stateful) do **Data Plane** (Pesado, Stateless).

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  VPS Control Plane                          │
│                  (Low Cost: $35/month)                      │
│                                                             │
│  ┌─────────────────┐         ┌──────────────┐              │
│  │ Airflow/Temporal│         │  Redis Queue │              │
│  │   (Scheduler)   │────────▶│              │              │
│  └─────────────────┘         └──────┬───────┘              │
│           │                         │                      │
│           │ Enfileira Job           │ Consome              │
│           │                         │                      │
│           ▼                         ▼                      │
│  ┌─────────────────────────────────────────────┐           │
│  │                                             │           │
│  │           Compute Data Plane                │           │
│  │          (On-Demand Instances)              │           │
│  │                                             │           │
│  │  ┌──────────────────┐  ┌─────────────────┐ │           │
│  │  │ Spot Instance G  │  │ Spot Instance G │ │           │
│  │  │ Serverless G     │  │ Serverless G    │ │           │
│  │  │                  │  │                 │ │           │
│  │  │ FFmpeg + Whisper │  │ FFmpeg + Whisper│ │           │
│  │  └────────┬─────────┘  └────────┬────────┘ │           │
│  │           │ Status              │ Upload   │           │
│  └───────────┼─────────────────────┼──────────┘           │
│              │                     │                      │
│              ▼                     ▼                      │
│         ┌─────────┐          ┌──────────┐                 │
│         │Dashboard│          │ Storage  │                 │
│         │   API   │          │ (Object) │                 │
│         └─────────┘          └──────────┘                 │
│              │                                             │
│         ┌────┴────┐                                        │
│         │PostgreSQL│                                       │
│         └─────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Control Plane (VPS - Sempre Ligado)

**Componentes:**
- **Airflow/Temporal:** Scheduler de workflows
- **Redis Queue:** Fila de jobs de renderização
- **Dashboard API:** Interface web (Next.js)
- **PostgreSQL:** Estado, metadata, analytics

**Características:**
- **Stateful:** Mantém estado de todos os vídeos
- **Low CPU:** Apenas orquestra, não renderiza
- **Sempre on:** Aceita requests 24/7

**Custo:** $35/mês (VPS fixo)

---

## Data Plane (On-Demand - Liga/Desliga)

**Componentes:**
- **Spot Instances / Serverless Compute**
- **FFmpeg + Whisper + MoviePy**
- **Apenas processamento pesado**

**Características:**
- **Stateless:** Não mantém nada, recebe job → processa → envia resultado
- **High CPU:** Toda a renderização acontece aqui
- **On-demand:** Liga quando há jobs na fila, desliga quando vazia

**Opções de Deployment:**

### Opção A: AWS Spot Instances
```yaml
Instance Type: c6i.xlarge (4 vCPU, 8GB RAM)
Cost: ~$0.05/hour (spot pricing)
Usage: 20 horas/mês (40 vídeos × 30 min)
Monthly Cost: $1.00
```

### Opção B: Google Cloud Run (Serverless)
```yaml
CPU: 4 vCPU
Memory: 8GB
Timeout: 30 min (por job)
Cost: $0.10/min de CPU ativo
Usage: 40 vídeos × 20 min = 800 min
Monthly Cost: $80
```

### Opção C: Hetzner Cloud (Budget)
```yaml
Instance: CPX21 (3 vCPU, 4GB RAM)
Cost: €0.01/hour (~$0.011/hour)
Usage: On-demand, liga apenas durante renderização
Monthly Cost: ~$3-5
```

**Recomendação:** Hetzner Cloud (melhor custo/benefício)

---

## Workflow Split-Plane

### 1. Job Creation (Control Plane)
```
User submits topic via Dashboard
↓
Airflow creates workflow
↓
Step 1-5: Research, Script, Audio, B-roll (Control Plane)
↓
Checkpoint 3: Approved
↓
Enqueue "render_job" in Redis
```

### 2. Job Processing (Data Plane)
```
Spot Instance checks Redis Queue
↓
Consome job (video_id, paths to audio/broll/subtitles)
↓
FFmpeg assembly (20 min)
↓
Upload final video to Object Storage
↓
Update PostgreSQL: status = "completed", video_path = "s3://..."
↓
Shutdown instance (or pick next job)
```

### 3. Status Updates (Control Plane)
```
Dashboard polls PostgreSQL every 5s
↓
Displays: "Rendering... 60% complete"
↓
When status = "completed" → Trigger Checkpoint 5
```

---

## Benefits

### Cost Optimization
```
Traditional (Always-On VPS):
  - VPS rendering 24/7: $100/month
  
Split-Plane:
  - Control Plane: $35/month (always on)
  - Data Plane: $5/month (20 hours on-demand)
  - Total: $40/month
  
Savings: 60%
```

### Performance
- **No blocking:** Renderização não trava API/Dashboard
- **Parallel processing:** 3+ vídeos simultaneamente (multi-instance)
- **Scalability:** Add more instances if queue grows

### Reliability
- **Isolation:** Crash em renderização não afeta Control Plane
- **Retry logic:** Job falha → Re-enqueue automatically
- **State preservation:** PostgreSQL sempre acessível

---

## Implementation Details

### Redis Queue Structure
```json
{
  "job_id": "uuid",
  "video_id": "uuid",
  "priority": 1,
  "created_at": "2026-02-03T10:00:00Z",
  "payload": {
    "audio_path": "s3://bucket/audio.mp3",
    "broll_clips": ["s3://bucket/clip1.mp4", ...],
    "subtitles_path": "s3://bucket/subs.srt",
    "config": {...}
  }
}
```

### Spot Instance Startup Script
```bash
#!/bin/bash
# Worker startup (Hetzner Cloud)

# Install dependencies
apt-get update && apt-get install -y ffmpeg python3 pip

# Clone worker code
git clone https://github.com/youraccount/yt-auto-worker.git
cd yt-auto-worker

# Install Python deps
pip install -r requirements.txt

# Start worker (connects to Redis)
python worker.py \
  --redis-url=$REDIS_URL \
  --postgres-url=$DATABASE_URL \
  --s3-bucket=$S3_BUCKET
```

### Worker Loop (Python)
```python
import redis
import subprocess

redis_client = redis.Redis.from_url(REDIS_URL)

while True:
    # Blocking pop (waits for job)
    job = redis_client.blpop('render_queue', timeout=60)
    
    if job is None:
        # No jobs for 60s → Shutdown
        shutdown_instance()
        break
    
    video_id, payload = parse_job(job)
    
    try:
        # Download assets from S3
        download_assets(payload)
        
        # Render with FFmpeg
        render_video(payload)
        
        # Upload result to S3
        upload_video(video_id)
        
        # Update database
        update_status(video_id, 'completed')
        
    except Exception as e:
        # Re-queue job with retry count
        requeue_job(job, retry_count=job.retry + 1)
        log_error(video_id, e)
```

---

## Auto-Scaling Logic

### Scale-Up Trigger
```
IF redis_queue.length > 5:
    AND no_idle_instances:
        Launch new Spot Instance
```

### Scale-Down Trigger
```
IF redis_queue.length == 0:
    FOR 5 minutes:
        Shutdown idle instance
```

### Max Instances
```
Limit: 3 concurrent instances
(Prevents cost explosion, 3 × $0.01/hour = $0.03/hour max)
```

---

## Monitoring

### Control Plane Metrics
- API response time (p95)
- PostgreSQL connections
- Redis queue length
- Dashboard uptime

### Data Plane Metrics
- Jobs processed / hour
- Average render time
- Failed jobs (retry rate)
- Cost per video

### Alerts
- **Critical:** Queue length > 10 (backlog)
- **Warning:** Render time > 30 min (performance issue)
- **Info:** Daily cost summary

---

## Migration Path

### Phase 1: Monolithic (Week 1-4)
- Everything on single VPS
- Validate pipeline works end-to-end

### Phase 2: Hybrid Prep (Week 5)
- Add Redis queue
- Refactor render as separate job
- Test local render worker

### Phase 3: Split-Plane (Week 6+)
- Deploy Control Plane on VPS
- Deploy Worker on Spot Instance
- Monitor cost savings

---

**This architecture scales cost-efficiently from 1 to 100+ canais.**

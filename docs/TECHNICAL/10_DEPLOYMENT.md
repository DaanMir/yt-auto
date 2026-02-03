# 10 - Deployment & Infraestrutura

## Visão Geral

Três opções de deploy, da mais simples à mais robusta:

| Opção | Quando usar | Custo |
|-------|------------|-------|
| Local | Desenvolvimento e testes | $0 |
| Single VPS | Produção até 20 canais | ~$35/mês |
| Kubernetes | 20+ canais, enterprise | $100+/mês |

**Recomendado para início:** Single VPS

## Opção A: Desenvolvimento Local

### Requisitos

- Python 3.10+
- Node.js 18+
- PostgreSQL 14+
- FFmpeg
- 8 GB RAM mínimo
- 50 GB storage livre

### Setup Local

```bash
# 1. Clonar repositório
git clone https://github.com/DaanMir/yt-auto.git
cd yt-auto

# 2. Setup Python
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows
pip install -r requirements.txt

# 3. Setup Node.js (Dashboard)
cd dashboard
npm install
cd ..

# 4. Setup PostgreSQL
createdb yt_auto
psql yt_auto < migrations/001_initial_schema.sql

# 5. Configurar environment
cp .env.example .env
# Editar .env com API keys

# 6. Instalar FFmpeg
# Ubuntu: sudo apt install ffmpeg
# Mac: brew install ffmpeg
# Windows: choco install ffmpeg

# 7. Instalar Whisper
pip install openai-whisper

# 8. Rodar
python main.py &           # Backend
cd dashboard && npm run dev # Frontend
```

### Estrutura .env

```env
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/yt_auto

# API Keys
GROQ_API_KEY=gsk_...
AZURE_TTS_KEY=...
AZURE_TTS_REGION=eastus
SERPER_API_KEY=...
PEXELS_API_KEY=...
PIXABAY_API_KEY=...

# YouTube OAuth
YOUTUBE_CLIENT_ID=...
YOUTUBE_CLIENT_SECRET=...

# Dashboard
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000/ws

# Optional
REDIS_URL=redis://localhost:6379
CANVA_API_KEY=...

# App Config
LOG_LEVEL=INFO
ENVIRONMENT=development
MAX_CONCURRENT_WORKFLOWS=3
```

## Opção B: Single VPS (Produção)

### Specs Recomendadas

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4 GB | 8 GB |
| Storage | 100 GB SSD | 200 GB SSD |
| Bandwidth | 2 TB/mês | 5 TB/mês |

**Providers sugeridos:**
- Hetzner (melhor custo/benefício): ~$15-35/mês
- DigitalOcean: ~$24-48/mês
- Contabo: ~$10-25/mês

### Docker Compose

```yaml
version: '3.8'

services:
  # Backend API
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db
      - redis
    volumes:
      - ./output:/app/output
      - ./config:/app/config
    restart: unless-stopped

  # Dashboard (Next.js)
  dashboard:
    build:
      context: ./dashboard
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - api
    restart: unless-stopped

  # Worker (background jobs)
  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    env_file:
      - .env
    depends_on:
      - db
      - redis
    volumes:
      - ./output:/app/output
      - ./config:/app/config
    restart: unless-stopped

  # PostgreSQL
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: yt_auto
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    restart: unless-stopped

  # Redis (cache opcional)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data
    restart: unless-stopped

  # Nginx (reverse proxy)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - api
      - dashboard
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

### Dockerfile (Backend)

```dockerfile
FROM python:3.11-slim

# Instalar FFmpeg
RUN apt-get update && apt-get install -y ffmpeg && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Instalar Whisper
RUN pip install openai-whisper

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Nginx Config

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # Dashboard
    location / {
        proxy_pass http://dashboard:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # API
    location /api/ {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket
    location /ws {
        proxy_pass http://api:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    # Static files (videos, thumbnails)
    location /output/ {
        alias /app/output/;
        expires 7d;
    }
}
```

### Deploy Commands

```bash
# 1. Setup VPS
ssh root@your-vps

# 2. Instalar Docker
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin

# 3. Clonar repo
git clone https://github.com/DaanMir/yt-auto.git
cd yt-auto

# 4. Configurar
cp .env.example .env
nano .env  # Editar com valores de produção

# 5. SSL (Let's Encrypt)
apt install certbot
certbot certonly --standalone -d yourdomain.com
cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem nginx/ssl/cert.pem
cp /etc/letsencrypt/live/yourdomain.com/privkey.pem nginx/ssl/key.pem

# 6. Build e start
docker compose up -d --build

# 7. Verificar
docker compose ps
docker compose logs -f api
```

## Opção C: Kubernetes (Enterprise)

Para 20+ canais, usar Kubernetes com auto-scaling:

```yaml
# Resumo da arquitetura K8s
# Deployments: api, dashboard, worker (com HPA)
# StatefulSets: postgresql
# Services: ClusterIP para internal, LoadBalancer para external
# Ingress: nginx-ingress com TLS
# PVC: para output files e PostgreSQL data
```

Detalhamento de K8s só é necessário se escalar além de 20 canais.

## Monitoramento

### Logs

```python
import structlog

logger = structlog.get_logger()

# Cada operação tem correlation_id
logger.info("workflow.step.completed",
    workflow_id=workflow.id,
    step="audio_generation",
    duration_seconds=45.2,
    correlation_id=workflow.correlation_id
)
```

### Health Checks

```python
# GET /api/v1/health
@app.get("/api/v1/health")
async def health_check():
    checks = {
        "api": "ok",
        "database": await check_db(),
        "redis": await check_redis(),
        "ffmpeg": check_ffmpeg(),
        "disk_space_gb": get_free_disk_space(),
    }
    status = "healthy" if all(v == "ok" or isinstance(v, (int, float)) for v in checks.values()) else "degraded"
    return {"status": status, "checks": checks}
```

### Alertas

| Alerta | Condição | Ação |
|--------|---------|------|
| Disk full | <10 GB livre | Limpar output antigos |
| DB connection lost | Health check fail | Restart container |
| API key expired | 401 de serviço externo | Renovar key |
| High error rate | >5% workflows failing | Investigar logs |
| Checkpoint backlog | >20 pendentes | Notificar humano |

## Backup

```bash
# Backup PostgreSQL (diário, via cron)
0 3 * * * docker exec db pg_dump -U postgres yt_auto | gzip > /backups/db_$(date +\%Y\%m\%d).sql.gz

# Backup output files (semanal)
0 4 * * 0 tar czf /backups/output_$(date +\%Y\%m\%d).tar.gz /app/output/

# Cleanup backups antigos (manter 30 dias)
0 5 * * * find /backups -mtime +30 -delete
```

## Storage Management

```python
class StorageManager:
    def cleanup_old_files(self, days_old: int = 30):
        """Remove arquivos intermediários de vídeos já publicados."""

        published_videos = self.db.query("""
            SELECT video_id, audio_path, video_path
            FROM videos
            WHERE status = 'published'
            AND published_at < NOW() - INTERVAL '%s days'
        """, (days_old,))

        for video in published_videos:
            # Manter: video final + thumbnail
            # Remover: audio, b-roll intermediários, subtitles
            self._remove_intermediary_files(video)

        # Reportar espaço liberado
        freed = self._calculate_freed_space()
        logger.info("storage.cleanup", freed_gb=freed)
```

## Custos Mensais (Single VPS)

| Item | Custo |
|------|-------|
| VPS (Hetzner CX31) | ~$15 |
| Serper API | ~$50 |
| Canva Pro | $13 |
| Domain + SSL | ~$1 |
| **Total** | **~$79/mês** |

---

**Anterior:** [09_WORKFLOW_CHECKPOINTS.md](./09_WORKFLOW_CHECKPOINTS.md)
**Próximo:** [12_IMPLEMENTATION_ROADMAP.md](./12_IMPLEMENTATION_ROADMAP.md)

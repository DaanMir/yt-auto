# 08 - API Specifications

## Visão Geral

REST API entre o Dashboard (Next.js) e o Backend (Python). Comunicação em tempo real via WebSocket para updates de progresso.

**Base URL:** `http://localhost:8000/api/v1`

## Autenticação

```
Authorization: Bearer <jwt_token>
```

### Endpoints de Auth

```
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

## Endpoints

### Channels

```
GET    /api/v1/channels                    # Listar canais
GET    /api/v1/channels/:channel_id        # Detalhes do canal
POST   /api/v1/channels                    # Criar canal
PUT    /api/v1/channels/:channel_id        # Atualizar canal
DELETE /api/v1/channels/:channel_id        # Remover canal
GET    /api/v1/channels/:channel_id/stats  # Estatísticas do canal
```

#### GET /api/v1/channels

```json
// Response 200
{
  "channels": [
    {
      "channel_id": "UC...",
      "channel_name": "Finanças Descomplicadas",
      "niche_id": "finance",
      "niche_name": "Finanças Pessoais",
      "is_active": true,
      "stats": {
        "total_videos": 45,
        "total_shorts": 180,
        "avg_ctr": 7.2,
        "videos_this_month": 8
      }
    }
  ],
  "total": 10
}
```

#### POST /api/v1/channels

```json
// Request
{
  "channel_name": "Tech Simplified",
  "niche_id": "tech",
  "youtube_handle": "@techsimplified"
}

// Response 201
{
  "channel_id": "UC...",
  "message": "Channel created. OAuth authorization required.",
  "oauth_url": "https://accounts.google.com/o/oauth2/..."
}
```

### Videos

```
GET    /api/v1/videos                          # Listar vídeos
GET    /api/v1/videos/:video_id                # Detalhes do vídeo
POST   /api/v1/videos                          # Criar vídeo (iniciar produção)
DELETE /api/v1/videos/:video_id                # Cancelar vídeo
GET    /api/v1/videos/:video_id/status         # Status do workflow
GET    /api/v1/videos/:video_id/script         # Script do vídeo
GET    /api/v1/videos/:video_id/analytics      # Analytics do vídeo
```

#### POST /api/v1/videos

```json
// Request
{
  "channel_id": "UC...",
  "topic": "Como investir em renda fixa em 2026",
  "priority": 5,
  "notes": "Focar em Tesouro Direto e CDB"
}

// Response 202
{
  "video_id": "550e8400-e29b-41d4-a716-446655440000",
  "workflow_id": "660e8400-e29b-41d4-a716-446655440001",
  "status": "researching",
  "message": "Production workflow started"
}
```

#### GET /api/v1/videos?status=ready&channel_id=UC...

```json
// Response 200
{
  "videos": [
    {
      "video_id": "550e8400-...",
      "title": "5 Investimentos em Renda Fixa para 2026",
      "topic": "Como investir em renda fixa em 2026",
      "video_type": "longform",
      "status": "ready",
      "channel_id": "UC...",
      "niche_id": "finance",
      "duration_seconds": 612,
      "word_count": 1850,
      "created_at": "2026-02-01T10:00:00Z",
      "scheduled_for": "2026-02-04T12:00:00Z"
    }
  ],
  "total": 12,
  "page": 1,
  "per_page": 20
}
```

### Checkpoints

```
GET    /api/v1/checkpoints                              # Listar pendentes
GET    /api/v1/checkpoints/:checkpoint_id               # Detalhes
POST   /api/v1/checkpoints/:checkpoint_id/approve       # Aprovar
POST   /api/v1/checkpoints/:checkpoint_id/reject        # Rejeitar
POST   /api/v1/checkpoints/:checkpoint_id/revision      # Pedir revisão
GET    /api/v1/checkpoints/batch                        # Listar para batch review
POST   /api/v1/checkpoints/batch/approve                # Aprovar em lote
```

#### GET /api/v1/checkpoints?status=pending

```json
// Response 200
{
  "checkpoints": [
    {
      "checkpoint_id": "770e8400-...",
      "workflow_id": "660e8400-...",
      "video_id": "550e8400-...",
      "video_title": "5 Investimentos em Renda Fixa para 2026",
      "checkpoint_type": "script_review",
      "status": "pending",
      "created_at": "2026-02-01T10:30:00Z",
      "data": {
        "script_preview": "Primeiras 500 palavras...",
        "word_count": 1850,
        "quality_score": 8.5,
        "issues": []
      }
    }
  ],
  "total": 8
}
```

#### POST /api/v1/checkpoints/:id/approve

```json
// Request
{
  "notes": "Aprovado. Bom script.",
  "modifications": null
}

// Response 200
{
  "checkpoint_id": "770e8400-...",
  "status": "approved",
  "next_step": "audio_generation",
  "message": "Checkpoint approved. Pipeline continuing."
}
```

#### POST /api/v1/checkpoints/:id/revision

```json
// Request
{
  "notes": "Adicionar mais dados sobre Tesouro IPCA+",
  "revision_type": "content_addition"
}

// Response 200
{
  "checkpoint_id": "770e8400-...",
  "status": "revision_requested",
  "message": "Revision requested. Pipeline will re-execute research step."
}
```

### Knowledge

```
GET    /api/v1/knowledge/facts?niche_id=finance         # Listar facts
GET    /api/v1/knowledge/facts/:fact_id                  # Detalhe do fact
POST   /api/v1/knowledge/query                           # Query knowledge
GET    /api/v1/knowledge/coverage?niche_id=finance       # Coverage tracker
GET    /api/v1/knowledge/health?niche_id=finance         # Health report
```

#### POST /api/v1/knowledge/query

```json
// Request
{
  "niche_id": "finance",
  "query": "renda fixa 2026",
  "max_results": 20
}

// Response 200
{
  "confidence": 0.72,
  "strategy": "partial_research",
  "facts": [
    {
      "fact_id": "...",
      "fact_text": "Taxa Selic atual: 13,25%",
      "confidence": 0.95,
      "source_name": "BCB",
      "expires_at": "2026-03-15T00:00:00Z"
    }
  ],
  "learnings": [
    {
      "learning_text": "Títulos com números geram +23% CTR",
      "type": "title",
      "confidence": 0.85
    }
  ],
  "coverage": [
    {
      "topic": "Tesouro Direto para iniciantes",
      "published_at": "2026-01-15T00:00:00Z",
      "video_type": "longform"
    }
  ],
  "gaps": ["taxas atualizadas fev 2026", "novos produtos renda fixa"]
}
```

### Analytics

```
GET    /api/v1/analytics/dashboard                       # Dashboard summary
GET    /api/v1/analytics/channels/:channel_id            # Analytics por canal
GET    /api/v1/analytics/niches/:niche_id                # Analytics por nicho
GET    /api/v1/analytics/costs                           # Custos operacionais
```

#### GET /api/v1/analytics/dashboard

```json
// Response 200
{
  "period": "2026-02",
  "overview": {
    "total_channels": 10,
    "active_channels": 10,
    "videos_produced": 38,
    "shorts_produced": 185,
    "total_views": 125000,
    "avg_ctr": 7.1,
    "total_cost": 87.50,
    "cost_per_video": 0.39
  },
  "channels": [
    {
      "channel_id": "UC...",
      "channel_name": "Finanças Descomplicadas",
      "videos_this_month": 4,
      "shorts_this_month": 20,
      "views": 15000,
      "avg_ctr": 7.8
    }
  ],
  "pending_checkpoints": 5,
  "active_workflows": 3,
  "knowledge_stats": {
    "total_facts": 2500,
    "reuse_rate": 0.73,
    "facts_expiring_soon": 45
  }
}
```

### Workflows

```
GET    /api/v1/workflows                                 # Listar workflows
GET    /api/v1/workflows/:workflow_id                    # Detalhes
POST   /api/v1/workflows/:workflow_id/retry              # Retry workflow
POST   /api/v1/workflows/:workflow_id/cancel             # Cancelar
```

### Niches

```
GET    /api/v1/niches                                    # Listar nichos
GET    /api/v1/niches/:niche_id                          # Detalhes
POST   /api/v1/niches                                    # Criar nicho
PUT    /api/v1/niches/:niche_id                          # Atualizar config
```

## WebSocket API

**URL:** `ws://localhost:8000/ws`

### Events (Server → Client)

```json
// Workflow progress
{
  "event": "workflow.progress",
  "data": {
    "workflow_id": "660e8400-...",
    "video_id": "550e8400-...",
    "step": "audio_generation",
    "progress": 65,
    "message": "Generating audio narration..."
  }
}

// Checkpoint created
{
  "event": "checkpoint.created",
  "data": {
    "checkpoint_id": "770e8400-...",
    "checkpoint_type": "script_review",
    "video_title": "5 Investimentos...",
    "pending_count": 6
  }
}

// Workflow completed
{
  "event": "workflow.completed",
  "data": {
    "workflow_id": "660e8400-...",
    "video_id": "550e8400-...",
    "video_title": "5 Investimentos...",
    "youtube_url": "https://youtube.com/watch?v=..."
  }
}

// Workflow error
{
  "event": "workflow.error",
  "data": {
    "workflow_id": "660e8400-...",
    "step": "broll_search",
    "error": "No relevant B-roll found",
    "retry_count": 2,
    "max_retries": 3
  }
}
```

### Events (Client → Server)

```json
// Subscribe to channel updates
{
  "action": "subscribe",
  "channels": ["UC...", "UC..."]
}

// Unsubscribe
{
  "action": "unsubscribe",
  "channels": ["UC..."]
}
```

## Error Responses

```json
// 400 Bad Request
{
  "error": "validation_error",
  "message": "Invalid niche_id",
  "details": {"field": "niche_id", "value": "invalid"}
}

// 401 Unauthorized
{
  "error": "unauthorized",
  "message": "Invalid or expired token"
}

// 404 Not Found
{
  "error": "not_found",
  "message": "Video not found",
  "resource": "video",
  "id": "550e8400-..."
}

// 429 Rate Limited
{
  "error": "rate_limited",
  "message": "Too many requests",
  "retry_after": 60
}

// 500 Internal Error
{
  "error": "internal_error",
  "message": "An unexpected error occurred",
  "trace_id": "abc123"
}
```

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| General | 100 req/min |
| Video creation | 10 req/min |
| Checkpoint actions | 50 req/min |
| Knowledge queries | 30 req/min |
| Analytics | 60 req/min |

## Pagination

Todos os endpoints de listagem suportam:

```
?page=1&per_page=20&sort=created_at&order=desc
```

---

**Anterior:** [07_DATABASE_SCHEMA.md](./07_DATABASE_SCHEMA.md)
**Próximo:** [09_WORKFLOW_CHECKPOINTS.md](./09_WORKFLOW_CHECKPOINTS.md)

# Arquitetura Técnica - Visão Geral

## Princípios Arquiteturais

### 1. Separation of Concerns
Cada nicho opera independentemente sem shared state.

### 2. Knowledge-First
Query knowledge base antes de qualquer external API call.

### 3. Human-in-the-Loop
Checkpoints estratégicos, não micro-management.

### 4. Fail-Safe
Erros em um canal não afetam outros.

### 5. Observable
Logs estruturados, métricas em tempo real.

## High-Level Architecture

```
┌─────────────────────────────────────────────┐
│           WEB DASHBOARD (UI)                │
│     Next.js + React + WebSocket             │
└─────────────────────────────────────────────┘
                    ↓ REST API
┌─────────────────────────────────────────────┐
│        ORCHESTRATION LAYER                  │
│     Airflow / Temporal (Workflow)           │
└─────────────────────────────────────────────┘
                    ↓
     ┌──────────────┴──────────────┐
     ↓                              ↓
┌──────────────┐           ┌──────────────┐
│ NICHE ROUTER │           │  KNOWLEDGE   │
│   (Dispatch) │←─────────→│    SYSTEM    │
└──────────────┘           └──────────────┘
     ↓                              ↑
     ↓ (Per Niche)                  ↑
┌──────────────────────────────────┐↑
│  PRODUCTION PIPELINE             ││
│  ┌────────────────────────────┐  ││
│  │ Research System            │──┘│
│  ├────────────────────────────┤   │
│  │ Long-form Pipeline         │   │
│  ├────────────────────────────┤   │
│  │ Shorts Pipeline            │   │
│  ├────────────────────────────┤   │
│  │ Upload System              │   │
│  └────────────────────────────┘   │
└───────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│           PERSISTENCE LAYER                 │
│  PostgreSQL + File Storage + Cache          │
└─────────────────────────────────────────────┘
```

## Camadas do Sistema

### Layer 1: Presentation (Dashboard)
**Responsabilidade:** Interface para operação humana

**Componentes:**
- Web UI (Next.js)
- Checkpoint interfaces
- Analytics dashboard
- Content calendar
- Real-time updates (WebSocket)

**Não contém:** Lógica de negócio, apenas apresentação

### Layer 2: Orchestration
**Responsabilidade:** Coordenar workflows, gerenciar estado

**Componentes:**
- Workflow engine (Airflow ou Temporal)
- Job scheduler
- State machine (checkpoints)
- Error handling & retry logic

**Decisões:**
- Qual nicho processar
- Quando executar jobs
- Como lidar com falhas
- Priorização de tarefas

### Layer 3: Niche Routing
**Responsabilidade:** Dispatcher para nicho correto

**Função:**
```
Input: channel_id, operation
Process:
  1. Lookup channel.niche
  2. Load niche config
  3. Initialize niche-specific components
  4. Execute operation
Output: Result OR error
```

**Crítico:** Zero leakage entre nichos

### Layer 4: Knowledge System
**Responsabilidade:** Armazenar e servir conhecimento acumulado

**Componentes:**
- Facts Database (dados factuais)
- Learnings Database (validações)
- Coverage Tracker (o que foi coberto)
- Query Engine (busca inteligente)

**Operações:**
- Query before research
- Store after production
- Consolidate periodicamente
- Expire outdated facts

### Layer 5: Production Pipelines
**Responsabilidade:** Produzir conteúdo (long-form + Shorts)

**Sub-sistemas:**
- Research Pipeline
- Long-form Pipeline
- Shorts Pipeline
- Upload System

**Cada um independente, stateless, observável**

### Layer 6: Persistence
**Responsabilidade:** Armazenamento durável

**Componentes:**
- PostgreSQL (metadata, state, analytics)
- File Storage (videos, audio, thumbnails)
- Cache (Redis opcional, para knowledge queries)

## Data Flow

### Long-form Production

```
1. USER: Submits topic via Dashboard
   ↓
2. ORCHESTRATOR: Creates workflow job
   ↓
3. NICHE ROUTER: Identifies niche, loads config
   ↓
4. KNOWLEDGE SYSTEM: Query existing knowledge
   ↓
5. RESEARCH PIPELINE:
   - If knowledge sufficient → Use cached
   - If knowledge gaps → Web research
   - Store new facts
   ↓
6. CHECKPOINT 1: Script review (HUMAN)
   ↓
7. AUDIO PIPELINE: Generate audio
   ↓
8. CHECKPOINT 2: Audio review (HUMAN)
   ↓
9. BROLL PIPELINE: Search + download
   ↓
10. CHECKPOINT 3: B-roll review (HUMAN)
    ↓
11. ASSEMBLY PIPELINE: Compose video
    ↓
12. THUMBNAIL PIPELINE: Generate options
    ↓
13. CHECKPOINT 4: Final review (HUMAN)
    ↓
14. UPLOAD SYSTEM: Schedule YouTube upload
    ↓
15. KNOWLEDGE SYSTEM: Extract & store learnings
    ↓
16. ANALYTICS: Track performance
```

### Shorts Production (Extraction)

```
1. TRIGGER: Long-form video completed
   ↓
2. SHORTS EXTRACTOR: Analyze script, identify segments
   ↓
3. SHORTS FORMATTER: Extract clips, reformat 9:16
   ↓
4. SHORTS ASSEMBLER: Add captions, optimize
   ↓
5. UPLOAD SYSTEM: Auto-schedule (no checkpoint)
```

## Component Communication

### Synchronous (REST API)
- Dashboard ↔ Orchestrator
- Orchestrator ↔ Knowledge System
- Quick queries, immediate response

### Asynchronous (Message Queue)
- Orchestrator → Production Pipelines
- Production → Knowledge System (updates)
- Long-running tasks, eventual consistency

### Event-Driven
- Video completed → Trigger Shorts extraction
- Checkpoint approved → Continue pipeline
- Error occurred → Notify + retry logic

## Error Handling Strategy

### Levels

**Level 1: Component Retry**
- Componente falha → Retry 3x com backoff
- Example: API call timeout

**Level 2: Workflow Retry**
- Workflow step falha → Retry step específico
- Example: B-roll search sem resultados

**Level 3: Human Intervention**
- Retry falhou → Marca para review manual
- Notifica via dashboard

**Level 4: Fail & Continue**
- Canal falha → Outros canais continuam
- Isolation prevents cascade

### Error Categories

**Transient (retry automático):**
- Network timeouts
- Rate limits
- Temporary API unavailability

**Permanent (human fix):**
- Invalid config
- Missing API keys
- Content policy violation

**Data Quality (checkpoint):**
- Low confidence research
- Poor B-roll relevance
- Script quality issues

## Scalability Considerations

### Horizontal Scaling
- **Stateless components:** Research, Assembly pipelines
- **Parallel execution:** 10 canais = 10 parallel jobs
- **Load balancing:** Distribute across workers

### Vertical Scaling
- **CPU-intensive:** Video assembly (FFmpeg)
- **Memory-intensive:** Whisper transcription
- **GPU optional:** Faster Whisper processing

### Database Scaling
- **Read replicas:** Analytics queries
- **Partitioning:** Por channel_id ou niche
- **Caching:** Frequent knowledge queries (Redis)

## Security Considerations

### API Keys
- **Storage:** Environment variables ou secrets manager
- **Rotation:** Automated key rotation
- **Scope:** Minimal permissions per service

### Data Privacy
- **User data:** GDPR compliant (EU users)
- **Video content:** No PII in automated content
- **Logs:** Anonymized, retention limits

### Access Control
- **Dashboard:** Role-based (admin, editor, viewer)
- **API:** Token-based authentication
- **YouTube:** OAuth2 per channel

## Monitoring & Observability

### Metrics to Track
- Production rate (videos/day)
- Checkpoint duration (avg)
- Failed jobs (%)
- API latency (p50, p95, p99)
- Knowledge reuse rate (%)
- Cost per video ($)

### Logging
- Structured logs (JSON)
- Correlation IDs (trace requests)
- Error categorization
- Performance profiling

### Alerting
- Critical: API down, upload failed
- Warning: Slow performance, low quality score
- Info: Daily summary, weekly report

## Technology Choices

### Required
- **Language:** Python 3.10+ (pipelines)
- **Database:** PostgreSQL 14+
- **Video:** FFmpeg, MoviePy
- **LLM:** Groq API (free tier)
- **TTS:** Azure TTS (free tier)
- **Dashboard:** Next.js, React

### Optional
- **Orchestration:** Airflow (complex) OU Temporal (simpler) OU custom (simplest)
- **Cache:** Redis (se knowledge queries lentas)
- **Queue:** RabbitMQ (se message volume alto)

### Deployment
- **Option A:** Local (dev/test)
- **Option B:** Single VPS (production até 20 canais)
- **Option C:** Kubernetes (20+ canais, enterprise)

---

**Arquitetura projetada para: Simplicidade inicial, escalabilidade futura.**

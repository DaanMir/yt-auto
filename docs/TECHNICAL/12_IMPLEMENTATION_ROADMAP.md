# 12 - Implementation Roadmap

## Visão Geral

Roadmap de implementação em 4 fases, priorizando o pipeline core (long-form) primeiro, depois expandindo para Shorts, multi-channel e otimizações.

## Fase 1: Core Long-form Pipeline

**Foco:** Pipeline funcional para 1 canal, 1 nicho

### 1.1 - Setup do Projeto

```
Deliverables:
├── Estrutura de diretórios
├── requirements.txt / package.json
├── Docker Compose (dev)
├── PostgreSQL schema (migrations)
├── .env.example
└── README com setup instructions
```

**Tasks:**
- [ ] Criar estrutura de pastas do projeto
- [ ] Setup Python environment (venv, requirements)
- [ ] Setup PostgreSQL + rodar migrations iniciais
- [ ] Configurar Docker Compose para dev
- [ ] Criar .env.example com todas variáveis necessárias

### 1.2 - Knowledge System

```
Deliverables:
├── Facts CRUD (create, read, query, expire)
├── Coverage Tracker
├── Query Engine (busca por relevância)
└── Testes unitários
```

**Tasks:**
- [ ] Implementar Facts model + repository
- [ ] Implementar Coverage model + repository
- [ ] Implementar Query Engine com cálculo de confidence
- [ ] Implementar expiração automática de facts
- [ ] Testes unitários para cada componente

### 1.3 - Research Pipeline

```
Deliverables:
├── Serper API integration
├── Fact extraction (via Groq)
├── Fact validation (cross-reference)
├── Script generation (via Groq)
├── Quality check automático
└── Testes de integração
```

**Tasks:**
- [ ] Integrar Serper API (web search)
- [ ] Integrar Groq API (LLM)
- [ ] Implementar fact extraction
- [ ] Implementar fact validation
- [ ] Implementar script generation com prompts por nicho
- [ ] Implementar quality check automático
- [ ] Testes de integração (pipeline completo)

### 1.4 - Audio Pipeline

```
Deliverables:
├── Azure TTS integration
├── SSML builder (pausas, ênfases)
├── Audio file management
└── Testes
```

**Tasks:**
- [ ] Integrar Azure TTS
- [ ] Implementar SSML builder
- [ ] Gerenciamento de arquivos de audio
- [ ] Testes com diferentes vozes e configurações

### 1.5 - B-roll Pipeline

```
Deliverables:
├── Pexels API integration
├── Pixabay API integration
├── Visual marker extraction do script
├── Clip selection + download
└── Gap filling
```

**Tasks:**
- [ ] Integrar Pexels API (video search + download)
- [ ] Integrar Pixabay API (video search + download)
- [ ] Implementar extração de visual markers do script
- [ ] Implementar seleção inteligente de clips
- [ ] Implementar gap filling (cobertura visual completa)

### 1.6 - Video Assembly

```
Deliverables:
├── FFmpeg/MoviePy integration
├── Timeline builder (audio + video + subtitles)
├── Whisper subtitle generation
├── Video rendering
└── Output management
```

**Tasks:**
- [ ] Integrar FFmpeg/MoviePy
- [ ] Implementar Whisper para legendas
- [ ] Implementar timeline builder
- [ ] Implementar renderização final
- [ ] Gerenciamento de arquivos de output

### 1.7 - Workflow Engine

```
Deliverables:
├── State machine (workflow states)
├── Step executor (sequential)
├── Checkpoint system
├── Retry logic
├── Error handling
└── Testes end-to-end
```

**Tasks:**
- [ ] Implementar state machine do workflow
- [ ] Implementar executor de steps
- [ ] Implementar checkpoints (pause/resume)
- [ ] Implementar retry com backoff
- [ ] Implementar error handling (4 níveis)
- [ ] Teste end-to-end: topic → video final

### Milestone Fase 1
**Critério de sucesso:** Produzir 1 vídeo completo de 10 minutos, do tópico ao upload, com checkpoints funcionais.

---

## Fase 2: Shorts System

**Foco:** Produção de Shorts a partir de long-form

### 2.1 - Shorts Extraction

```
Deliverables:
├── Script analyzer (identificar segmentos)
├── Segment selector (ranking)
├── Hook rewriter
└── Testes
```

**Tasks:**
- [ ] Implementar análise de script para segmentos "shortáveis"
- [ ] Implementar ranking de segmentos por potencial viral
- [ ] Implementar reescrita de hook
- [ ] Testes com scripts reais

### 2.2 - Shorts Production

```
Deliverables:
├── 16:9 → 9:16 reformatter
├── Caption overlay (word-by-word)
├── Auto-scheduling
└── Testes
```

**Tasks:**
- [ ] Implementar reformatação visual (crop + blur background)
- [ ] Implementar legendas word-by-word estilo TikTok
- [ ] Implementar auto-scheduling de uploads
- [ ] Testes end-to-end: long-form → 3-5 Shorts

### Milestone Fase 2
**Critério de sucesso:** A partir de 1 long-form, extrair automaticamente 3-5 Shorts e agendá-los para upload.

---

## Fase 3: Multi-channel + Dashboard

**Foco:** Escalar para 10 canais com interface de controle

### 3.1 - Niche System

```
Deliverables:
├── Niche Router
├── Config loader (niche templates)
├── Isolation validation
├── 3+ niches configurados
└── Testes de isolamento
```

**Tasks:**
- [ ] Implementar Niche Router
- [ ] Implementar loader de configs por nicho
- [ ] Criar templates para 3+ nichos
- [ ] Testes de isolamento (zero cross-contamination)

### 3.2 - Multi-channel Support

```
Deliverables:
├── YouTube OAuth per channel
├── Parallel workflow execution
├── Channel isolation
└── Testes com 3+ canais
```

**Tasks:**
- [ ] Implementar OAuth2 para múltiplos canais YouTube
- [ ] Implementar execução paralela de workflows
- [ ] Garantir isolamento entre canais
- [ ] Testes com 3+ canais simultâneos

### 3.3 - Dashboard (MVP)

```
Deliverables:
├── Next.js project setup
├── Channel overview page
├── Checkpoint review interface
├── Video list + status
├── WebSocket real-time updates
└── Basic authentication
```

**Tasks:**
- [ ] Setup Next.js + React project
- [ ] Implementar página de overview (canais, stats)
- [ ] Implementar interface de checkpoint review
- [ ] Implementar lista de vídeos com filtros
- [ ] Implementar WebSocket para updates em tempo real
- [ ] Implementar autenticação básica (JWT)

### 3.4 - REST API

```
Deliverables:
├── FastAPI setup
├── Todos endpoints documentados em 08_API_SPECIFICATIONS
├── WebSocket server
├── Rate limiting
└── Error handling
```

**Tasks:**
- [ ] Setup FastAPI com auto-docs (Swagger)
- [ ] Implementar endpoints de channels
- [ ] Implementar endpoints de videos
- [ ] Implementar endpoints de checkpoints (incluindo batch)
- [ ] Implementar endpoints de knowledge
- [ ] Implementar endpoints de analytics
- [ ] Implementar WebSocket server
- [ ] Rate limiting e error handling

### Milestone Fase 3
**Critério de sucesso:** 3 canais de nichos diferentes produzindo conteúdo simultaneamente, gerenciados via Dashboard web.

---

## Fase 4: Optimization + Analytics

**Foco:** Melhorar qualidade, reduzir custos, adicionar analytics

### 4.1 - Thumbnail System

```
Deliverables:
├── Pillow-based thumbnail generator
├── Canva Pro integration (opcional)
├── 3 variantes por vídeo
├── Template system por nicho
└── Testes
```

**Tasks:**
- [ ] Implementar gerador de thumbnails com Pillow
- [ ] Implementar sistema de templates por nicho
- [ ] Implementar geração de 3 variantes
- [ ] Integrar Canva Pro API (opcional)

### 4.2 - Analytics System

```
Deliverables:
├── YouTube Analytics API integration
├── Daily snapshot collector
├── Performance dashboard
├── Cost tracking
├── Learnings extractor
└── Reports
```

**Tasks:**
- [ ] Integrar YouTube Analytics API
- [ ] Implementar collector de métricas diárias
- [ ] Implementar dashboard de performance
- [ ] Implementar tracking de custos
- [ ] Implementar extração automática de learnings
- [ ] Implementar relatórios semanais/mensais

### 4.3 - Knowledge Optimization

```
Deliverables:
├── Consolidation job (semanal)
├── Deduplication
├── Confidence recalculation
├── Health reports
└── Alertas de expiração
```

**Tasks:**
- [ ] Implementar job de consolidação semanal
- [ ] Implementar deduplicação de facts
- [ ] Implementar recálculo de confidence
- [ ] Implementar health reports
- [ ] Implementar alertas para facts expirando

### 4.4 - Production Deployment

```
Deliverables:
├── Docker production build
├── Nginx config + SSL
├── Backup automation
├── Monitoring + alertas
├── Storage cleanup
└── Documentação de operação
```

**Tasks:**
- [ ] Otimizar Dockerfiles para produção
- [ ] Configurar Nginx + Let's Encrypt
- [ ] Implementar backup automático (DB + files)
- [ ] Implementar health checks e alertas
- [ ] Implementar cleanup automático de storage

### Milestone Fase 4
**Critério de sucesso:** Sistema rodando em VPS com 10 canais, analytics funcionais, custo <$100/mês, <3 horas/semana de operação humana.

---

## Dependências entre Componentes

```
Knowledge System ←── Research Pipeline ←── Script Generation
                                              ↓
                                        Audio Pipeline
                                              ↓
                                        B-roll Pipeline
                                              ↓
                                       Video Assembly
                                              ↓
                                       Thumbnail System
                                              ↓
                                        Upload System
                                              ↓
                                       Analytics System
```

## Ordem de Implementação Recomendada

```
1. Database Schema (07)
2. Knowledge System (02)
3. Niche System (01) - config básico
4. Research Pipeline (03)
5. Audio Pipeline (parte de 04)
6. B-roll Pipeline (parte de 04)
7. Video Assembly (parte de 04)
8. Workflow Engine (09)
9. Shorts Pipeline (05)
10. Thumbnail System (06)
11. API (08)
12. Dashboard (parte de 08)
13. Deployment (10)
14. Analytics + Optimization (este doc)
```

## Tech Stack Final

| Componente | Tecnologia |
|------------|-----------|
| Backend | Python 3.11 + FastAPI |
| Frontend | Next.js 14 + React |
| Database | PostgreSQL 16 |
| Cache | Redis 7 |
| Video | FFmpeg + MoviePy |
| TTS | Azure TTS (free tier) |
| LLM | Groq API (free tier) |
| Search | Serper API |
| Stock Media | Pexels + Pixabay |
| Subtitles | Whisper (local) |
| Thumbnails | Pillow + Canva Pro |
| Deploy | Docker Compose + Nginx |
| CI/CD | GitHub Actions |

---

**Anterior:** [10_DEPLOYMENT.md](./10_DEPLOYMENT.md)
**Primeiro:** [00_ARCHITECTURE.md](./00_ARCHITECTURE.md)

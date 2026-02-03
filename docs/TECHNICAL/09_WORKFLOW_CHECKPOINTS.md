# 09 - Workflow & Checkpoints

## Visão Geral

O sistema de workflows coordena a produção de conteúdo, gerenciando estado, retries e checkpoints humanos. Cada vídeo é produzido por um workflow que avança através de steps definidos.

**Princípio:** Human-in-the-Loop nos pontos críticos, não micro-management.

## State Machine

### Estados do Workflow

```
         ┌──────────┐
         │ PENDING  │
         └────┬─────┘
              ↓
         ┌──────────┐
    ┌───→│ RUNNING  │←──────────────┐
    │    └────┬─────┘               │
    │         │                     │
    │         ↓                     │
    │    ┌──────────┐          ┌────┴─────┐
    │    │  PAUSED  │──────────│ RESUMED  │
    │    │(checkpoint)         └──────────┘
    │    └────┬─────┘
    │         │ (rejected)
    │         ↓
    │    ┌──────────┐
    │    │ REVISION │──────────→ (volta para step anterior)
    │    └──────────┘
    │
    │    ┌──────────┐          ┌──────────┐
    └────│ RETRYING │          │COMPLETED │
         └──────────┘          └──────────┘
              ↑                     ↑
              │                     │
         ┌────┴─────┐              │
         │  FAILED  │──────────────┘
         └──────────┘  (retry succeeds)
```

### Estados do Video

```
draft → researching → scripted → producing → assembling → ready → uploaded → published
                                                                       ↓
                                                                   failed
```

## Workflow Engine

### Definição do Workflow

```python
class LongformWorkflow:
    """Workflow completo para produção de vídeo long-form."""

    STEPS = [
        Step("knowledge_query",   handler=knowledge_query,   checkpoint=False),
        Step("research",          handler=research,           checkpoint=False),
        Step("script_generation", handler=generate_script,    checkpoint=False),
        Step("script_review",     handler=None,               checkpoint=True),   # CHECKPOINT 1
        Step("audio_generation",  handler=generate_audio,     checkpoint=False),
        Step("audio_review",      handler=None,               checkpoint=True),   # CHECKPOINT 2
        Step("broll_search",      handler=search_broll,       checkpoint=False),
        Step("broll_review",      handler=None,               checkpoint=True),   # CHECKPOINT 3
        Step("video_assembly",    handler=assemble_video,     checkpoint=False),
        Step("subtitle_gen",      handler=generate_subtitles, checkpoint=False),
        Step("thumbnail_gen",     handler=generate_thumbnail, checkpoint=False),
        Step("final_review",      handler=None,               checkpoint=True),   # CHECKPOINT 4
        Step("upload",            handler=upload_video,        checkpoint=False),
        Step("knowledge_store",   handler=store_knowledge,    checkpoint=False),
        Step("analytics_init",    handler=init_analytics,     checkpoint=False),
    ]

    def execute(self, video_id: str):
        workflow = self._create_workflow(video_id)

        for step in self.STEPS:
            try:
                workflow.current_step = step.name
                self._update_status(workflow, "running")

                if step.checkpoint:
                    # Pausar e aguardar aprovação humana
                    self._create_checkpoint(workflow, step)
                    self._update_status(workflow, "paused")
                    return  # Retorna. Será retomado via API.

                # Executar step
                result = step.handler(workflow.context)
                workflow.context.update(result)

                self._log_step_complete(workflow, step)

            except TransientError as e:
                self._handle_retry(workflow, step, e)
            except PermanentError as e:
                self._handle_failure(workflow, step, e)

        self._update_status(workflow, "completed")
```

### Retomada após Checkpoint

```python
class WorkflowResumer:
    def resume(self, checkpoint_id: str, action: str, notes: str = None):
        """Retoma workflow após decisão humana no checkpoint."""

        checkpoint = self.db.get_checkpoint(checkpoint_id)
        workflow = self.db.get_workflow(checkpoint.workflow_id)

        if action == "approved":
            checkpoint.status = "approved"
            checkpoint.reviewer_notes = notes
            checkpoint.reviewed_at = datetime.now()

            # Continuar do próximo step
            next_step_index = self._get_step_index(workflow.current_step) + 1
            self._execute_from_step(workflow, next_step_index)

        elif action == "rejected":
            checkpoint.status = "rejected"
            checkpoint.reviewer_notes = notes

            # Marcar workflow como failed
            workflow.status = "failed"
            workflow.error_message = f"Rejected at {checkpoint.checkpoint_type}: {notes}"

        elif action == "revision_requested":
            checkpoint.status = "revision_requested"
            checkpoint.reviewer_notes = notes

            # Voltar para step de geração
            revision_step = self._get_revision_step(checkpoint.checkpoint_type)
            self._execute_from_step(workflow, revision_step)

        self.db.save(checkpoint)
        self.db.save(workflow)
```

## Checkpoints Detalhados

### CHECKPOINT 1: Script Review

**Quando:** Após pesquisa e geração de script
**O que o humano vê:**

```json
{
  "checkpoint_type": "script_review",
  "data": {
    "topic": "Como investir em renda fixa em 2026",
    "script_full": "...",
    "word_count": 1850,
    "quality_report": {
      "passed": true,
      "readability_score": 8.5,
      "visual_markers": 8,
      "issues": []
    },
    "research_sources": [
      {"name": "Investopedia", "url": "...", "confidence": 0.95}
    ],
    "facts_used": 12,
    "knowledge_reuse_rate": 0.65
  }
}
```

**Ações possíveis:**
- **Approve** → Continua para audio generation
- **Revision** → Re-genera script com feedback (volta ao research se necessário)
- **Reject** → Cancela produção deste vídeo

### CHECKPOINT 2: Audio Review

**Quando:** Após geração de audio
**O que o humano vê:**

```json
{
  "checkpoint_type": "audio_review",
  "data": {
    "audio_url": "/api/v1/files/audio/550e8400.wav",
    "duration_seconds": 612,
    "voice_used": "azure_pt_br_antonio",
    "script_text": "... (para acompanhar)"
  }
}
```

**Ações possíveis:**
- **Approve** → Continua para B-roll search
- **Revision** → Re-gera audio (ajustar velocidade, voz, etc.)
- **Reject** → Cancela

### CHECKPOINT 3: B-roll Review

**Quando:** Após busca de B-roll
**O que o humano vê:**

```json
{
  "checkpoint_type": "broll_review",
  "data": {
    "clips": [
      {
        "thumbnail_url": "/api/v1/files/broll/clip_001_thumb.jpg",
        "preview_url": "/api/v1/files/broll/clip_001.mp4",
        "source": "pexels",
        "query_used": "stock market graph",
        "duration_seconds": 15,
        "relevance_score": 0.85
      }
    ],
    "total_clips": 12,
    "total_duration": 620,
    "audio_duration": 612,
    "coverage_percentage": 98
  }
}
```

**Ações possíveis:**
- **Approve** → Continua para assembly
- **Revision** → Re-busca clips específicos (humano indica quais substituir)
- **Reject** → Cancela

### CHECKPOINT 4: Final Review

**Quando:** Após assembly completo (video + thumbnail)
**O que o humano vê:**

```json
{
  "checkpoint_type": "final_review",
  "data": {
    "video_preview_url": "/api/v1/files/video/550e8400_final.mp4",
    "duration_seconds": 612,
    "resolution": "1920x1080",
    "file_size_mb": 380,
    "thumbnail_variants": [
      {"url": "/api/v1/files/thumb/variant_1.png", "text": "5X MAIS"},
      {"url": "/api/v1/files/thumb/variant_2.png", "text": "RENDA FIXA"},
      {"url": "/api/v1/files/thumb/variant_3.png", "text": "INVISTA CERTO"}
    ],
    "metadata": {
      "title": "5 Investimentos em Renda Fixa para 2026",
      "description": "...",
      "tags": ["finanças", "renda fixa", "investimentos"]
    },
    "scheduled_for": "2026-02-04T12:00:00Z"
  }
}
```

**Ações possíveis:**
- **Approve** (com thumbnail selecionada) → Upload ao YouTube
- **Revision** → Re-assembly ou re-thumbnail
- **Reject** → Cancela

## Batch Checkpoints

Para eficiência, o humano pode revisar múltiplos checkpoints de uma vez:

```
GET /api/v1/checkpoints/batch?type=script_review&limit=10
```

```json
{
  "batch": [
    {"checkpoint_id": "...", "video_title": "...", "script_preview": "..."},
    {"checkpoint_id": "...", "video_title": "...", "script_preview": "..."},
    // ... até 10
  ],
  "total_pending": 8
}
```

```
POST /api/v1/checkpoints/batch/approve
{
  "checkpoint_ids": ["...", "...", "..."],
  "notes": "Batch approved"
}
```

## Error Handling no Workflow

### Retry Strategy

```python
RETRY_CONFIG = {
    "knowledge_query":   {"max_retries": 3, "backoff": "exponential", "base_delay": 2},
    "research":          {"max_retries": 3, "backoff": "exponential", "base_delay": 5},
    "script_generation": {"max_retries": 2, "backoff": "fixed",       "base_delay": 10},
    "audio_generation":  {"max_retries": 3, "backoff": "exponential", "base_delay": 5},
    "broll_search":      {"max_retries": 3, "backoff": "exponential", "base_delay": 5},
    "video_assembly":    {"max_retries": 1, "backoff": "fixed",       "base_delay": 30},
    "thumbnail_gen":     {"max_retries": 2, "backoff": "fixed",       "base_delay": 5},
    "upload":            {"max_retries": 5, "backoff": "exponential", "base_delay": 60},
}

def _handle_retry(self, workflow, step, error):
    config = RETRY_CONFIG[step.name]

    if workflow.retry_count >= config["max_retries"]:
        self._handle_failure(workflow, step, error)
        return

    workflow.retry_count += 1

    if config["backoff"] == "exponential":
        delay = config["base_delay"] * (2 ** workflow.retry_count)
    else:
        delay = config["base_delay"]

    self._schedule_retry(workflow, step, delay)
```

### Isolation entre Canais

```python
def execute_channel_batch(self, channel_ids: list[str]):
    """Executa workflows para múltiplos canais em paralelo, com isolation."""

    for channel_id in channel_ids:
        try:
            self._process_channel(channel_id)
        except Exception as e:
            # Falha em um canal NÃO afeta outros
            logger.error(f"Channel {channel_id} failed: {e}")
            self._notify_failure(channel_id, e)
            continue  # Próximo canal
```

## Métricas de Workflow

| Métrica | Descrição | Meta |
|---------|-----------|------|
| Avg checkpoint wait | Tempo médio esperando aprovação | <2 horas |
| Approval rate | % de checkpoints aprovados na primeira vez | >85% |
| Retry rate | % de steps que precisaram retry | <10% |
| End-to-end time | Tempo total do workflow (excl. checkpoint wait) | <30 min |
| Failure rate | % de workflows que falharam permanentemente | <2% |

---

**Anterior:** [08_API_SPECIFICATIONS.md](./08_API_SPECIFICATIONS.md)
**Próximo:** [10_DEPLOYMENT.md](./10_DEPLOYMENT.md)

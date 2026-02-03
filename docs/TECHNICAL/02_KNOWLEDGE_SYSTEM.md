# 02 - Knowledge System

## Visão Geral

O Knowledge System é o diferencial competitivo do sistema. Ele acumula, organiza e reutiliza conhecimento entre produções, gerando **economia de 70-90% em research** após os primeiros meses.

**Princípio:** Query knowledge base ANTES de qualquer API call externa.

## Arquitetura do Knowledge System

```
┌─────────────────────────────────────────────┐
│            KNOWLEDGE SYSTEM                 │
│                                             │
│  ┌───────────┐  ┌───────────┐  ┌─────────┐ │
│  │   Facts   │  │ Learnings │  │Coverage │ │
│  │ Database  │  │ Database  │  │ Tracker │ │
│  └─────┬─────┘  └─────┬─────┘  └────┬────┘ │
│        │              │              │      │
│  ┌─────┴──────────────┴──────────────┴────┐ │
│  │          QUERY ENGINE                  │ │
│  │   (Busca inteligente por relevância)   │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Componentes

### 1. Facts Database

Armazena dados factuais verificados, organizados por nicho.

```sql
CREATE TABLE facts (
    fact_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    fact_text        TEXT NOT NULL,
    source_url      TEXT,
    source_name     VARCHAR(200),
    confidence      DECIMAL(3,2) DEFAULT 0.80,  -- 0.00 a 1.00
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP,                   -- NULL = nunca expira
    used_count      INTEGER DEFAULT 0,
    last_used_at    TIMESTAMP,
    verified_by     VARCHAR(50),                 -- 'human' ou 'auto'
    tags            TEXT[],

    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id)
);

CREATE INDEX idx_facts_niche ON facts(niche_id);
CREATE INDEX idx_facts_category ON facts(niche_id, category);
CREATE INDEX idx_facts_tags ON facts USING GIN(tags);
CREATE INDEX idx_facts_expires ON facts(expires_at) WHERE expires_at IS NOT NULL;
```

**Exemplos de Facts:**

| Nicho | Categoria | Fact | Expira? |
|-------|-----------|------|---------|
| finance | taxa_selic | "Taxa Selic atual: 13,25% (Jan 2026)" | 45 dias |
| tech | specs | "iPhone 16 tem chip A18 Pro" | Nunca |
| health | nutrition | "Vitamina D recomendada: 600-800 UI/dia" | 365 dias |

### 2. Learnings Database

Armazena lições aprendidas sobre o que funciona (ou não) em conteúdo.

```sql
CREATE TABLE learnings (
    learning_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    type            VARCHAR(50) NOT NULL,  -- 'content', 'thumbnail', 'hook', 'title'
    learning_text   TEXT NOT NULL,
    evidence        JSONB,                 -- dados que suportam o learning
    confidence      DECIMAL(3,2) DEFAULT 0.70,
    validated       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW(),
    video_id        VARCHAR(20),           -- video que gerou o learning

    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id)
);
```

**Exemplos de Learnings:**

```json
{
  "niche_id": "finance",
  "type": "title",
  "learning_text": "Títulos com números específicos geram 23% mais CTR que títulos genéricos",
  "evidence": {
    "sample_size": 15,
    "avg_ctr_with_numbers": 7.2,
    "avg_ctr_without_numbers": 5.8
  },
  "confidence": 0.85,
  "validated": true
}
```

### 3. Coverage Tracker

Registra quais tópicos já foram cobertos para evitar repetição.

```sql
CREATE TABLE coverage (
    coverage_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    channel_id      VARCHAR(50) NOT NULL,
    topic           VARCHAR(300) NOT NULL,
    subtopics       TEXT[],
    video_id        VARCHAR(20),
    video_type      VARCHAR(20),  -- 'longform' ou 'short'
    published_at    TIMESTAMP,
    performance     JSONB,        -- views, ctr, retention

    CONSTRAINT fk_channel FOREIGN KEY (channel_id)
        REFERENCES channels(channel_id)
);

CREATE INDEX idx_coverage_topic ON coverage(niche_id, topic);
CREATE INDEX idx_coverage_channel ON coverage(channel_id);
```

### 4. Query Engine

Busca inteligente que combina facts, learnings e coverage para responder queries.

```python
class KnowledgeQueryEngine:
    def query(self, niche_id: str, query: str, context: dict = None) -> QueryResult:
        """
        Busca knowledge relevante antes de fazer research externo.

        Returns:
            QueryResult com:
            - facts: lista de fatos relevantes
            - learnings: lições aplicáveis
            - coverage: tópicos já cobertos (para evitar repetição)
            - confidence: quão completa é a resposta
            - gaps: o que falta pesquisar
        """
        # 1. Buscar facts relevantes
        facts = self._search_facts(niche_id, query)

        # 2. Buscar learnings aplicáveis
        learnings = self._search_learnings(niche_id, query)

        # 3. Verificar coverage
        coverage = self._check_coverage(niche_id, query)

        # 4. Calcular confiança
        confidence = self._calculate_confidence(facts, learnings, query)

        # 5. Identificar gaps
        gaps = self._identify_gaps(facts, query, context)

        return QueryResult(
            facts=facts,
            learnings=learnings,
            coverage=coverage,
            confidence=confidence,
            gaps=gaps
        )

    def _calculate_confidence(self, facts, learnings, query) -> float:
        """
        Confidence >= 0.8 → Usar knowledge cache (sem research externo)
        Confidence 0.5-0.8 → Research parcial (só gaps)
        Confidence < 0.5 → Research completo
        """
        if not facts:
            return 0.0

        avg_fact_confidence = sum(f.confidence for f in facts) / len(facts)
        coverage_score = min(len(facts) / 5, 1.0)  # 5+ facts = full coverage
        recency_score = self._recency_score(facts)

        return (avg_fact_confidence * 0.4 +
                coverage_score * 0.4 +
                recency_score * 0.2)
```

## Fluxo de Operação

### Query Before Research

```
1. TOPIC: "Como investir em renda fixa em 2026"
   ↓
2. QUERY ENGINE: Busca knowledge
   ↓
3. RESULT:
   - 12 facts sobre renda fixa (Selic, CDB, Tesouro)
   - 3 learnings (títulos que funcionam, duração ideal)
   - 2 vídeos já cobriram renda fixa (evitar repetição)
   - Confidence: 0.72
   - Gaps: ["taxas atualizadas 2026", "novos produtos"]
   ↓
4. DECISION: Research parcial (só gaps)
   ↓
5. RESEARCH: Busca apenas taxas 2026 e novos produtos
   ↓
6. STORE: Novos facts armazenados para uso futuro
```

### Store After Production

Após cada vídeo produzido, o sistema extrai e armazena:

```python
def extract_and_store(self, video_data: dict):
    """Extrai knowledge de um vídeo produzido."""

    # 1. Extrair facts do script
    facts = self.llm.extract_facts(video_data["script"])
    for fact in facts:
        self.facts_db.store(fact)

    # 2. Registrar coverage
    self.coverage_tracker.register(
        channel_id=video_data["channel_id"],
        topic=video_data["topic"],
        subtopics=video_data["subtopics"],
        video_id=video_data["youtube_id"]
    )

    # 3. Após analytics disponível, extrair learnings
    # (scheduled job, roda 7 dias após publicação)
```

### Consolidação Periódica

Job semanal que mantém a knowledge base saudável:

```python
def consolidate(self, niche_id: str):
    """Manutenção semanal da knowledge base."""

    # 1. Expirar facts antigos
    self.facts_db.expire_old_facts(niche_id)

    # 2. Mergear facts duplicados
    self.facts_db.deduplicate(niche_id)

    # 3. Validar learnings com novos dados
    self.learnings_db.revalidate(niche_id)

    # 4. Atualizar confidence scores
    self.facts_db.recalculate_confidence(niche_id)

    # 5. Gerar relatório
    return self.generate_health_report(niche_id)
```

## Métricas do Knowledge System

| Métrica | Descrição | Meta |
|---------|-----------|------|
| Reuse Rate | % de research que usou cache | >70% após 3 meses |
| Cache Hit Rate | % de queries com confidence >= 0.8 | >50% após 3 meses |
| Fact Count | Total de facts por nicho | 500+ por nicho |
| Freshness | % de facts não expirados | >90% |
| Cost Savings | $ economizado vs research completo | >60% |

## Regras de Expiração

| Tipo de Fact | TTL | Exemplo |
|-------------|-----|---------|
| Taxas/Preços | 30-90 dias | Taxa Selic, preço Bitcoin |
| Specs técnicas | Nunca | Chip do iPhone, RAM |
| Dados científicos | 365 dias | Estudos de nutrição |
| Eventos | 7 dias | Notícias, lançamentos |
| Conceitos | Nunca | O que é CDI, O que é RAM |

---

**Anterior:** [01_NICHE_SYSTEM.md](./01_NICHE_SYSTEM.md)
**Próximo:** [03_RESEARCH_PIPELINE.md](./03_RESEARCH_PIPELINE.md)

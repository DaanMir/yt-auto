# 03 - Research Pipeline

## Visão Geral

O Research Pipeline é responsável por transformar um tópico em um script completo e factualmente correto, usando o Knowledge System para minimizar chamadas externas.

## Fluxo Completo

```
TOPIC INPUT
   ↓
┌──────────────────────┐
│ 1. Knowledge Query   │ ← Query Engine (02_KNOWLEDGE_SYSTEM)
│    Confidence?       │
└──────┬───────────────┘
       │
       ├─ >= 0.8 → Usar cache (sem web research)
       ├─ 0.5-0.8 → Research parcial (só gaps)
       └─ < 0.5 → Research completo
       │
┌──────┴───────────────┐
│ 2. Web Research      │ ← Serper API (se necessário)
│    (Gaps only)       │
└──────┬───────────────┘
       │
┌──────┴───────────────┐
│ 3. Fact Extraction   │ ← LLM extrai facts dos resultados
└──────┬───────────────┘
       │
┌──────┴───────────────┐
│ 4. Fact Validation   │ ← Cross-reference entre fontes
└──────┬───────────────┘
       │
┌──────┴───────────────┐
│ 5. Script Generation │ ← LLM gera script com facts validados
└──────┬───────────────┘
       │
┌──────┴───────────────┐
│ 6. Quality Check     │ ← Validação automática
└──────┬───────────────┘
       │
┌──────┴───────────────┐
│ 7. Store Knowledge   │ ← Salva novos facts no Knowledge System
└──────┬───────────────┘
       │
   CHECKPOINT 1: Script Review (HUMAN)
```

## Etapas Detalhadas

### 1. Knowledge Query

```python
class ResearchPipeline:
    def research(self, topic: str, niche_id: str, channel_id: str) -> ResearchResult:
        # 1. Query knowledge base
        knowledge = self.knowledge_engine.query(
            niche_id=niche_id,
            query=topic,
            context={"channel_id": channel_id}
        )

        # 2. Decidir estratégia
        if knowledge.confidence >= 0.8:
            strategy = "cache_only"
            web_queries = []
        elif knowledge.confidence >= 0.5:
            strategy = "partial_research"
            web_queries = self._generate_gap_queries(knowledge.gaps)
        else:
            strategy = "full_research"
            web_queries = self._generate_full_queries(topic, niche_id)

        return self._execute_strategy(strategy, knowledge, web_queries, topic, niche_id)
```

### 2. Web Research (Serper API)

```python
class WebResearcher:
    def __init__(self):
        self.serper_api_key = os.getenv("SERPER_API_KEY")
        self.base_url = "https://google.serper.dev/search"

    def search(self, queries: list[str], niche_config: dict) -> list[SearchResult]:
        results = []
        for query in queries:
            response = requests.post(
                self.base_url,
                json={"q": query, "num": 10, "gl": "br", "hl": "pt"},
                headers={"X-API-KEY": self.serper_api_key}
            )
            raw_results = response.json().get("organic", [])

            # Filtrar por fontes confiáveis do nicho
            filtered = self._filter_sources(
                raw_results,
                trusted=niche_config["research_config"]["trusted_sources"],
                blocked=niche_config["research_config"]["blocked_sources"]
            )
            results.extend(filtered)

        return results

    def _filter_sources(self, results, trusted, blocked):
        """Prioriza fontes confiáveis, bloqueia fontes ruins."""
        filtered = []
        for r in results:
            domain = urlparse(r["link"]).netloc
            if domain in blocked:
                continue
            r["trust_score"] = 1.0 if domain in trusted else 0.5
            filtered.append(r)
        return sorted(filtered, key=lambda x: x["trust_score"], reverse=True)
```

### 3. Fact Extraction

```python
def extract_facts(self, search_results: list[SearchResult], topic: str) -> list[Fact]:
    """Usa LLM para extrair fatos dos resultados de pesquisa."""

    prompt = f"""
    Analise os seguintes resultados de pesquisa sobre "{topic}".
    Extraia APENAS fatos verificáveis e objetivos.

    Para cada fato, forneça:
    - fact_text: O fato em uma frase clara
    - source: URL da fonte
    - confidence: 0.0-1.0 (quão confiável é)
    - category: Categoria do fato
    - expires: Quando este fato pode ficar desatualizado (null se permanente)

    Resultados:
    {self._format_results(search_results)}

    Responda em JSON.
    """

    response = self.groq_client.chat(prompt)
    return self._parse_facts(response)
```

### 4. Fact Validation

```python
def validate_facts(self, facts: list[Fact], niche_id: str) -> list[Fact]:
    """Cross-reference facts entre múltiplas fontes."""

    validated = []
    for fact in facts:
        # Verificar se fact aparece em múltiplas fontes
        corroboration = self._count_corroboration(fact, facts)

        if corroboration >= 2:
            fact.confidence = min(fact.confidence + 0.15, 1.0)
            fact.verified_by = "auto_multi_source"
        elif corroboration == 1:
            fact.verified_by = "auto_single_source"
        else:
            fact.confidence *= 0.7  # Reduz confiança
            fact.verified_by = "unverified"

        # Verificar contra knowledge base existente
        existing = self.knowledge_engine.find_similar(niche_id, fact.fact_text)
        if existing and existing.contradicts(fact):
            fact.confidence *= 0.5
            fact.flag = "contradiction_detected"

        validated.append(fact)

    return validated
```

### 5. Script Generation

```python
def generate_script(self, topic: str, facts: list[Fact],
                     learnings: list[Learning], niche_config: dict) -> Script:
    """Gera script usando facts validados e learnings do nicho."""

    rules = niche_config["content_rules"]

    prompt = f"""
    Crie um script de vídeo sobre "{topic}" para YouTube.

    REGRAS DO NICHO:
    - Palavras: {rules['min_script_words']}-{rules['max_script_words']}
    - Seções obrigatórias: {rules['required_sections']}
    - Tom: {rules['tone']}
    - Público: {rules['target_audience']}
    - Disclaimers obrigatórios: {rules['required_disclaimers']}

    FATOS VERIFICADOS (USE APENAS ESTES):
    {self._format_facts(facts)}

    LEARNINGS DO NICHO (APLIQUE ESTAS LIÇÕES):
    {self._format_learnings(learnings)}

    ESTRUTURA:
    1. Hook (primeiros 30 segundos - prender atenção)
    2. Introdução do tema
    3. Conteúdo principal (3-5 pontos)
    4. Conclusão
    5. CTA (call to action)

    Responda com o script completo, marcando [VISUAL: descrição] para indicar B-roll.
    """

    response = self.groq_client.chat(prompt)
    return Script(
        text=response,
        topic=topic,
        facts_used=facts,
        word_count=len(response.split()),
        niche_id=niche_config["niche_id"]
    )
```

### 6. Quality Check Automático

```python
def quality_check(self, script: Script, niche_config: dict) -> QualityReport:
    """Validação automática antes do checkpoint humano."""

    issues = []
    rules = niche_config["content_rules"]

    # Word count
    if script.word_count < rules["min_script_words"]:
        issues.append(f"Script muito curto: {script.word_count} palavras (min: {rules['min_script_words']})")
    if script.word_count > rules["max_script_words"]:
        issues.append(f"Script muito longo: {script.word_count} palavras (max: {rules['max_script_words']})")

    # Seções obrigatórias
    for section in rules["required_sections"]:
        if not self._has_section(script.text, section):
            issues.append(f"Seção obrigatória ausente: {section}")

    # Disclaimers
    for disclaimer in rules.get("required_disclaimers", []):
        if disclaimer.lower() not in script.text.lower():
            issues.append(f"Disclaimer obrigatório ausente")

    # Tópicos proibidos
    for topic in rules.get("forbidden_topics", []):
        if topic.lower() in script.text.lower():
            issues.append(f"Tópico proibido detectado: {topic}")

    # Visual markers
    visual_count = script.text.count("[VISUAL:")
    if visual_count < 3:
        issues.append(f"Poucas marcações visuais: {visual_count} (min recomendado: 3)")

    return QualityReport(
        passed=len(issues) == 0,
        issues=issues,
        word_count=script.word_count,
        visual_markers=visual_count,
        readability_score=self._calculate_readability(script.text)
    )
```

### 7. Store Knowledge

Após o script ser aprovado no checkpoint:

```python
def store_new_knowledge(self, script: Script, facts: list[Fact]):
    """Armazena novos facts e registra coverage."""

    # Armazenar facts novos
    for fact in facts:
        if fact.verified_by != "unverified":
            self.knowledge_db.store_fact(fact)

    # Registrar coverage
    self.coverage_tracker.register(
        niche_id=script.niche_id,
        topic=script.topic,
        subtopics=self._extract_subtopics(script.text)
    )
```

## Custos de Research

| Operação | Serviço | Custo |
|----------|---------|-------|
| Web search | Serper API | $0.004/query |
| Fact extraction | Groq | Free |
| Script generation | Groq | Free |
| Quality check | Groq | Free |

**Custo médio por vídeo:**
- Research completo: ~$0.04 (10 queries)
- Research parcial: ~$0.012 (3 queries)
- Cache only: $0.00

**Com Knowledge System ativo (após 3 meses):**
- 70% cache only, 25% parcial, 5% completo
- Custo médio: ~$0.005/vídeo

## Handling de Erros

| Erro | Ação |
|------|------|
| Serper API timeout | Retry 3x com backoff |
| Serper rate limit | Queue e retry em 60s |
| Groq timeout | Retry 3x |
| Script quality fail | Re-generate com feedback |
| Zero search results | Ampliar query, remover filtros |
| Fact contradiction | Flag para review humano |

---

**Anterior:** [02_KNOWLEDGE_SYSTEM.md](./02_KNOWLEDGE_SYSTEM.md)
**Próximo:** [04_LONGFORM_PIPELINE.md](./04_LONGFORM_PIPELINE.md)

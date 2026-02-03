# 01 - Sistema de Nichos

## Conceito Central

Cada canal do YouTube pertence a exatamente **um nicho**. Nichos operam de forma totalmente independente, sem compartilhamento de estado, conhecimento ou configuração entre si.

**Regra #1: Zero cross-contamination** - Um Finance Specialist NUNCA valida conteúdo de Tech.

## Estrutura de um Nicho

```
config/niche_templates/
├── finance/
│   ├── niche.schema.json       # Configuração do nicho
│   ├── prompts/                # Prompts especializados
│   │   ├── research.txt
│   │   ├── script.txt
│   │   ├── thumbnail.txt
│   │   └── shorts_hook.txt
│   ├── style/                  # Estilo visual
│   │   ├── fonts.json
│   │   ├── colors.json
│   │   └── thumbnail_templates/
│   └── rules/                  # Regras de conteúdo
│       ├── forbidden_topics.json
│       ├── required_disclaimers.json
│       └── quality_criteria.json
├── tech/
│   └── ... (mesma estrutura)
└── health/
    └── ... (mesma estrutura)
```

## Schema de Configuração do Nicho

```json
{
  "niche_id": "finance",
  "display_name": "Finanças Pessoais",
  "language": "pt-BR",
  "channels": ["channel_001", "channel_002"],

  "content_rules": {
    "min_script_words": 1500,
    "max_script_words": 3000,
    "required_sections": ["intro", "main_content", "conclusion", "cta"],
    "forbidden_topics": ["crypto_trading_signals", "get_rich_quick"],
    "required_disclaimers": ["Este conteúdo é educacional, não é aconselhamento financeiro."],
    "tone": "professional_friendly",
    "target_audience": "25-45, classe média"
  },

  "research_config": {
    "trusted_sources": ["investopedia.com", "bcb.gov.br", "infomoney.com"],
    "blocked_sources": ["reddit.com/r/wallstreetbets"],
    "max_source_age_days": 90,
    "min_sources_per_topic": 3,
    "fact_check_required": true
  },

  "video_config": {
    "target_duration_seconds": 600,
    "min_duration_seconds": 480,
    "max_duration_seconds": 900,
    "aspect_ratio": "16:9",
    "resolution": "1920x1080",
    "fps": 30
  },

  "shorts_config": {
    "target_duration_seconds": 45,
    "max_duration_seconds": 58,
    "aspect_ratio": "9:16",
    "resolution": "1080x1920",
    "hooks_style": "question_first"
  },

  "audio_config": {
    "voice_id": "azure_pt_br_antonio",
    "speed": 1.0,
    "pitch": 0,
    "style": "narration-professional"
  },

  "thumbnail_config": {
    "style": "bold_text_gradient",
    "max_text_words": 5,
    "use_faces": false,
    "color_palette": ["#1a1a2e", "#16213e", "#0f3460", "#e94560"],
    "font_family": "Montserrat"
  },

  "upload_config": {
    "default_tags": ["finanças", "investimentos", "dinheiro"],
    "category_id": "22",
    "privacy": "public",
    "schedule_strategy": "optimal_time",
    "preferred_days": ["tuesday", "thursday", "saturday"],
    "preferred_hours": [8, 12, 18]
  }
}
```

## Niche Router

O Niche Router é o dispatcher central que garante que cada operação é executada no contexto correto.

### Fluxo do Router

```
Input: (channel_id, operation, payload)
   ↓
1. Lookup: channel_id → niche_id (via DB)
   ↓
2. Validate: niche_id exists in config
   ↓
3. Load: niche config + prompts + rules
   ↓
4. Initialize: niche-specific components
   ↓
5. Execute: operation with niche context
   ↓
Output: Result OR Error
```

### Implementação Conceitual

```python
class NicheRouter:
    def __init__(self, config_path: str):
        self.niches = self._load_all_niches(config_path)

    def route(self, channel_id: str, operation: str, payload: dict) -> Result:
        # 1. Resolve niche
        niche_id = self._get_niche_for_channel(channel_id)

        # 2. Load config
        niche_config = self.niches[niche_id]

        # 3. Create isolated context
        context = NicheContext(
            niche_id=niche_id,
            config=niche_config,
            channel_id=channel_id
        )

        # 4. Execute with isolation
        pipeline = PipelineFactory.create(operation, context)
        return pipeline.execute(payload)

    def _get_niche_for_channel(self, channel_id: str) -> str:
        """Lookup no banco - cada channel pertence a exatamente 1 niche."""
        result = db.query(
            "SELECT niche_id FROM channels WHERE channel_id = %s",
            (channel_id,)
        )
        if not result:
            raise ChannelNotFoundError(channel_id)
        return result.niche_id
```

## Isolamento entre Nichos

### O que é isolado

| Recurso | Isolamento | Motivo |
|---------|-----------|--------|
| Knowledge Base | Por nicho | Fatos de finanças não se aplicam a tech |
| Prompts | Por nicho | Tom e estilo diferentes |
| B-roll | Por nicho | Imagens contextuais diferentes |
| Thumbnails | Por nicho | Identidade visual distinta |
| Upload schedule | Por canal | Audiências diferentes |
| Analytics | Por canal | Métricas independentes |

### O que é compartilhado

| Recurso | Compartilhamento | Motivo |
|---------|-----------------|--------|
| Infrastructure | Global | Um PostgreSQL serve todos |
| FFmpeg/MoviePy | Global | Ferramentas de vídeo genéricas |
| API keys | Global | Uma conta Groq serve todos |
| Dashboard | Global | Interface unificada |

## Adicionando um Novo Nicho

### Checklist

1. Criar diretório em `config/niche_templates/<niche_id>/`
2. Criar `niche.schema.json` com todas as configurações
3. Escrever prompts especializados (research, script, thumbnail, shorts)
4. Definir regras de conteúdo (tópicos proibidos, disclaimers)
5. Configurar paleta visual (cores, fontes, templates)
6. Registrar no banco de dados
7. Associar canal(is) ao nicho
8. Testar pipeline completo com 1 vídeo

### Validação

```python
def validate_niche_config(config: dict) -> list[str]:
    """Retorna lista de erros. Lista vazia = config válida."""
    errors = []

    required_fields = [
        "niche_id", "display_name", "language",
        "content_rules", "research_config", "video_config",
        "audio_config", "thumbnail_config", "upload_config"
    ]

    for field in required_fields:
        if field not in config:
            errors.append(f"Missing required field: {field}")

    if config.get("content_rules", {}).get("min_script_words", 0) < 500:
        errors.append("min_script_words deve ser >= 500")

    if config.get("thumbnail_config", {}).get("use_faces", True):
        errors.append("use_faces deve ser false (melhor CTR sem rostos)")

    return errors
```

## Métricas por Nicho

Cada nicho mantém métricas independentes:

- **Production Rate**: Vídeos produzidos/mês
- **Knowledge Reuse**: % de research que usou knowledge cache
- **Quality Score**: Média dos scores de checkpoint
- **CTR**: Click-through rate médio no YouTube
- **Retention**: Retenção média dos vídeos
- **Cost per Video**: Custo médio de API calls por vídeo

---

**Próximo:** [02_KNOWLEDGE_SYSTEM.md](./02_KNOWLEDGE_SYSTEM.md) - Como o sistema acumula e reutiliza conhecimento

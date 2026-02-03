# YouTube Automation Factory v3.1

Sistema automatizado de produção de conteúdo para YouTube capaz de gerenciar 10+ canais simultaneamente.

## 🎯 Objetivo

Produzir 40 vídeos longos/mês + 200 Shorts/mês para 10 canais com:
- ~95% automação
- <3 horas/semana tempo humano
- $50-150/mês custo operacional
- Qualidade profissional (CTR 6-9%)

## 📚 Documentação

### Para Entendimento Geral
```
docs/COMPLETE/
├── 01_SYSTEM_OVERVIEW.md     # O que é e como funciona
├── 02_BUSINESS_MODEL.md      # ROI, custos, métricas
└── 03_USER_GUIDE.md          # Como operar o sistema
```

### Para Desenvolvimento
```
docs/TECHNICAL/
├── 00_ARCHITECTURE.md         # Arquitetura geral
├── 01_NICHE_SYSTEM.md         # Sistema de nichos
├── 02_KNOWLEDGE_SYSTEM.md     # Acúmulo de conhecimento
├── 03_RESEARCH_PIPELINE.md    # Pipeline de pesquisa
├── 04_LONGFORM_PIPELINE.md    # Pipeline vídeos longos
├── 05_SHORTS_PIPELINE.md      # Pipeline Shorts
├── 06_THUMBNAIL_SYSTEM.md     # Sistema de thumbnails
├── 07_DATABASE_SCHEMA.md      # Schema de banco de dados
├── 08_API_SPECIFICATIONS.md   # APIs e endpoints
├── 09_WORKFLOW_CHECKPOINTS.md # Checkpoints e validações
├── 10_DEPLOYMENT.md           # Deploy e infraestrutura
├── 11_QUALITY_FRAMEWORK.md    # Q-Score e validação de qualidade ⭐
├── 12_IMPLEMENTATION_ROADMAP.md # Roadmap de implementação
└── 13_SPLIT_PLANE_ARCHITECTURE.md # Arquitetura híbrida ⭐
```

## 🚀 Quick Start

```bash
# 1. Setup estrutura
./scripts/setup.sh

# 2. Configurar nichos
# Editar config/niche_templates/*.schema.json

# 3. Iniciar desenvolvimento
# Seguir docs/TECHNICAL/ na ordem
```

## 💡 Princípios de Design

1. **Niche-Specialized**: Cada nicho independente, zero cross-contamination
2. **Knowledge Accumulation**: Sistema aprende e reutiliza conhecimento
3. **Human-in-the-Loop**: Validação humana em pontos críticos
4. **Quality Assurance**: Q-Score framework garante qualidade escalável ⭐
5. **Cost-Conscious**: Split-Plane architecture otimiza custos ⭐
6. **Observable**: Fácil identificar e corrigir problemas

## 🎓 Ordem de Leitura

### Para Product Managers / Stakeholders
1. COMPLETE/01_SYSTEM_OVERVIEW.md
2. COMPLETE/02_BUSINESS_MODEL.md
3. TECHNICAL/11_QUALITY_FRAMEWORK.md ⭐
4. COMPLETE/03_USER_GUIDE.md

### Para Desenvolvedores
**Fundação (obrigatório):**
1. TECHNICAL/00_ARCHITECTURE.md
2. TECHNICAL/13_SPLIT_PLANE_ARCHITECTURE.md ⭐
3. TECHNICAL/01_NICHE_SYSTEM.md
4. TECHNICAL/02_KNOWLEDGE_SYSTEM.md

**Por Componente:**
5. TECHNICAL/03_RESEARCH_PIPELINE.md
6. TECHNICAL/04_LONGFORM_PIPELINE.md
7. TECHNICAL/05_SHORTS_PIPELINE.md
8. TECHNICAL/06_THUMBNAIL_SYSTEM.md
9. TECHNICAL/11_QUALITY_FRAMEWORK.md ⭐

**Implementação:**
10. TECHNICAL/07_DATABASE_SCHEMA.md
11. TECHNICAL/08_API_SPECIFICATIONS.md
12. TECHNICAL/09_WORKFLOW_CHECKPOINTS.md
13. TECHNICAL/10_DEPLOYMENT.md
14. TECHNICAL/12_IMPLEMENTATION_ROADMAP.md

## 📊 Stack Tecnológico

**Free Tier:**
- Groq (LLM)
- Azure TTS (Audio)
- Pexels/Pixabay (B-roll)
- Whisper (Subtitles)
- FFmpeg/MoviePy (Video)

**Paid:**
- Canva Pro ($13/mês)
- Serper API ($50/mês)
- VPS Control Plane ($35/mês) ⭐
- Spot Instances Data Plane ($5/mês on-demand) ⭐

**Total: ~$100/mês** (60% savings via Split-Plane)

## 📈 Roadmap

**Fase 1 (Semanas 1-2)**: Core pipeline long-form  
**Fase 2 (Semana 3)**: Shorts system  
**Fase 3 (Semana 4)**: Multi-channel + Dashboard  
**Fase 4 (Semanas 5-6)**: Optimization + Q-Score ⭐  
**Fase 5 (Pós-Launch)**: Split-Plane migration ⭐

## ⭐ Novas Features v3.1

### Q-Score Quality Framework
Sistema de validação automática que garante qualidade profissional em escala:
- **Tone Variance (30%)**: Análise de variação de pitch (Librosa)
- **Pacing Visual (30%)**: Scene change rate (FFmpeg)
- **Semantic Relevance (20%)**: CLIP Score (script vs B-roll)
- **Human Eval (20%)**: Checklist rápido (3 perguntas)

**Threshold:** Q-Score >= 7.0 para upload

**Ver:** `docs/TECHNICAL/11_QUALITY_FRAMEWORK.md`

### Split-Plane Architecture
Arquitetura híbrida que separa Control Plane (stateful, always-on) do Data Plane (stateless, on-demand):
- **Control Plane**: VPS $35/mês (API, DB, Scheduler)
- **Data Plane**: Spot Instances $5/mês (Rendering on-demand)
- **Savings**: 60% vs monolithic architecture
- **Scalability**: Auto-scaling baseado em queue length

**Ver:** `docs/TECHNICAL/13_SPLIT_PLANE_ARCHITECTURE.md`

### Niche Validation Protocol (NVP)
Processo de 3 fases para validar novos nichos antes de produção em massa:
1. **Sondagem (Week 1)**: 5 Shorts low-effort → Meta: 1K views
2. **Piloto (Week 2)**: 3 long-form quality → Meta: CTR 4%
3. **Scale-Up (Month 1)**: Produção completa se métricas atingidas

**Ver:** `docs/TECHNICAL/11_QUALITY_FRAMEWORK.md`

## ⚠️ Conceitos Críticos

### ❌ Erros Comuns a Evitar
1. **NÃO misturar nichos**: Finance Specialist NUNCA valida Tech content
2. **NÃO ignorar Knowledge System**: Economia de 70-90% em research
3. **NÃO pular checkpoints**: Qualidade depende de validação humana
4. **NÃO usar rostos**: Thumbnails funcionam melhor sem rostos
5. **NÃO rodar rendering 24/7 em VPS fixo**: Use Split-Plane ⭐

### ✅ Boas Práticas
1. **Separação total por nicho**: Diretórios, configs, specialists independentes
2. **Query knowledge antes research**: Economiza API calls e garante consistência
3. **Batch checkpoints**: Revisa 10 scripts de uma vez, não 1 por 1
4. **Shorts de long-form**: 70% dos Shorts vêm de extração
5. **Q-Score >= 7.0**: Não faça upload sem validar qualidade ⭐
6. **Validar nicho com NVP**: 5 Shorts → 3 pilots → Scale-up ⭐

## 🔧 Diferencial Competitivo

### vs Produção Manual
- ✅ 50x mais rápido
- ✅ 95% mais barato
- ✅ Qualidade auditável (Q-Score) ⭐

### vs SaaS Tools (Pictory, InVideo)
- ✅ Research de verdade (não scraping)
- ✅ Knowledge accumulation (aprende)
- ✅ Niche specialization (não genérico)
- ✅ Cost-optimized infrastructure (Split-Plane) ⭐

### vs Outros Automation
- ✅ Quality framework sistemático ⭐
- ✅ Hybrid architecture escalável ⭐
- ✅ Niche validation protocol ⭐

---

**Versão**: 3.1  
**Status**: Pronto para implementação  
**Última atualização**: Fevereiro 2026  
**Changelog v3.1**: Q-Score framework + Split-Plane architecture + NVP protocol

# 05 - Shorts Pipeline

## Visão Geral

O Shorts Pipeline produz vídeos curtos (15-58 segundos) em formato 9:16. **70% dos Shorts são extraídos de vídeos long-form**, os 30% restantes são criados de forma independente.

**Meta:** 200 Shorts/mês por canal (5 por dia)

## Estratégia de Produção

```
SHORTS SOURCES:
├── 70% Extraction (do long-form)
│   └── Triggered automaticamente após long-form completo
├── 20% Repurpose (re-edit de Shorts antigos com bom desempenho)
│   └── Triggered por analytics (CTR > 8%)
└── 10% Original (standalone)
    └── Triggered por topic queue
```

## Pipeline de Extração (70%)

```
TRIGGER: Long-form video completed
   ↓
┌──────────────────────┐
│ 1. Script Analysis   │ ← LLM identifica segmentos "shortáveis"
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 2. Segment Selection │ ← Ranqueia por potencial viral
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 3. Hook Rewrite      │ ← LLM reescreve início para hook forte
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 4. Audio Extraction  │ ← Recorta audio do segmento
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 5. Visual Reformat   │ ← 16:9 → 9:16, crop/reframe
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 6. Caption Overlay   │ ← Legendas animadas (estilo TikTok)
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 7. Auto Upload       │ ← Schedule sem checkpoint humano
└──────┘
```

### 1. Script Analysis

```python
class ShortsExtractor:
    def analyze_script(self, script: str, video_duration: float) -> list[ShortCandidate]:
        """Identifica segmentos do script que funcionam como Shorts."""

        prompt = f"""
        Analise o script abaixo e identifique segmentos que funcionariam
        como YouTube Shorts (15-58 segundos).

        Critérios para um bom Short:
        1. É auto-contido (faz sentido sozinho)
        2. Tem um fato surpreendente ou insight valioso
        3. Começa com potencial de hook forte
        4. Não depende de contexto anterior
        5. Tem conclusão satisfatória

        Script:
        {script}

        Para cada segmento, retorne:
        - start_marker: texto onde começa
        - end_marker: texto onde termina
        - hook_potential: 1-10 (quão "clicável" é)
        - topic_summary: resumo em 1 frase
        - estimated_duration: duração estimada em segundos

        Identifique no máximo 5 segmentos. Retorne JSON.
        """

        response = self.groq_client.chat(prompt)
        candidates = self._parse_candidates(response)

        # Filtrar por duração
        return [c for c in candidates
                if 15 <= c.estimated_duration <= 58]
```

### 2. Segment Selection

```python
def select_segments(self, candidates: list[ShortCandidate],
                     max_shorts: int = 3) -> list[ShortCandidate]:
    """Seleciona os melhores segmentos para Shorts."""

    # Ranquear por potencial
    ranked = sorted(candidates, key=lambda c: c.hook_potential, reverse=True)

    # Verificar diversidade (não pegar segmentos muito próximos)
    selected = []
    for candidate in ranked:
        if len(selected) >= max_shorts:
            break

        # Evitar overlap
        overlaps = any(
            self._segments_overlap(candidate, s)
            for s in selected
        )
        if not overlaps:
            selected.append(candidate)

    return selected
```

### 3. Hook Rewrite

```python
def rewrite_hook(self, segment: ShortCandidate, niche_config: dict) -> str:
    """Reescreve o início do segmento para hook mais forte."""

    hooks_style = niche_config["shorts_config"]["hooks_style"]

    prompt = f"""
    Reescreva o início deste segmento para funcionar como hook de YouTube Short.

    Estilo de hook: {hooks_style}

    Estilos possíveis:
    - question_first: Começa com pergunta provocativa
    - bold_claim: Começa com afirmação impactante
    - curiosity_gap: Cria curiosidade que só resolve no final
    - counter_intuitive: Apresenta algo contra-intuitivo

    Segmento original:
    {segment.text}

    Regras:
    - Hook deve ter no máximo 2 frases
    - Deve prender atenção em 2 segundos
    - Deve ser verdadeiro (não clickbait falso)
    - O resto do segmento permanece igual

    Retorne apenas o texto reescrito completo.
    """

    return self.groq_client.chat(prompt)
```

### 5. Visual Reformat (16:9 → 9:16)

```python
class ShortsVisualFormatter:
    def reformat(self, source_video: str, segment_start: float,
                  segment_end: float) -> str:
        """Converte clip 16:9 para 9:16 com crop inteligente."""

        output_path = f"output/shorts/{uuid4()}.mp4"

        # Estratégia: center crop com blur background
        ffmpeg_cmd = [
            "ffmpeg",
            "-i", source_video,
            "-ss", str(segment_start),
            "-t", str(segment_end - segment_start),
            "-filter_complex", (
                # Background: blur + scale to 9:16
                "[0:v]scale=1080:1920:force_original_aspect_ratio=increase,"
                "crop=1080:1920,boxblur=20:20[bg];"
                # Foreground: original video scaled to fit width
                "[0:v]scale=1080:-1[fg];"
                # Overlay centered
                "[bg][fg]overlay=(W-w)/2:(H-h)/2"
            ),
            "-c:v", "libx264",
            "-c:a", "aac",
            "-preset", "fast",
            output_path
        ]

        subprocess.run(ffmpeg_cmd, check=True)
        return output_path
```

### 6. Caption Overlay

```python
class CaptionOverlay:
    def add_captions(self, video_path: str, audio_path: str) -> str:
        """Adiciona legendas animadas estilo TikTok/Shorts."""

        # 1. Transcrever com Whisper (word-level timestamps)
        import whisper
        model = whisper.load_model("base")
        result = model.transcribe(audio_path, language="pt", word_timestamps=True)

        # 2. Gerar legendas word-by-word
        captions = []
        for segment in result["segments"]:
            for word in segment.get("words", []):
                captions.append({
                    "text": word["word"],
                    "start": word["start"],
                    "end": word["end"]
                })

        # 3. Renderizar com estilo
        output_path = video_path.replace(".mp4", "_captioned.mp4")

        # ASS subtitle com estilo animado
        ass_content = self._generate_ass_subtitles(
            captions,
            style={
                "font": "Montserrat",
                "font_size": 48,
                "bold": True,
                "color": "#FFFFFF",
                "outline_color": "#000000",
                "outline_width": 3,
                "position": "center",       # centro da tela
                "animation": "word_highlight" # destaca palavra atual
            }
        )

        # Burn subtitles no vídeo
        subprocess.run([
            "ffmpeg",
            "-i", video_path,
            "-vf", f"ass={ass_content}",
            "-c:v", "libx264",
            "-c:a", "copy",
            output_path
        ], check=True)

        return output_path
```

## Pipeline Original (10%)

Para Shorts standalone (não extraídos de long-form):

```python
class StandaloneShortsPipeline:
    def produce(self, topic: str, niche_id: str) -> ShortResult:
        """Produz Short original do zero."""

        niche_config = self.niche_router.get_config(niche_id)

        # 1. Gerar micro-script (50-100 palavras)
        script = self._generate_short_script(topic, niche_config)

        # 2. Gerar audio
        audio = self.audio_pipeline.generate_audio(
            script, niche_config,
            max_duration=niche_config["shorts_config"]["max_duration_seconds"]
        )

        # 3. Buscar B-roll vertical
        clips = self.broll_pipeline.find_broll(
            script,
            audio.duration,
            orientation="portrait"  # 9:16
        )

        # 4. Montar
        video = self._assemble_short(audio, clips, niche_config)

        # 5. Adicionar captions
        video = self.caption_overlay.add_captions(video, audio.file_path)

        return ShortResult(file_path=video, duration=audio.duration)
```

## Upload Schedule (Shorts)

```python
class ShortsScheduler:
    def schedule(self, shorts: list[ShortResult], channel_id: str,
                  niche_config: dict) -> list[ScheduledUpload]:
        """Distribui Shorts ao longo da semana."""

        # Meta: ~5 Shorts/dia, espaçados
        uploads_per_day = 5
        optimal_hours = [7, 10, 13, 17, 20]  # horários de pico

        scheduled = []
        current_date = datetime.now()

        for i, short in enumerate(shorts):
            day_offset = i // uploads_per_day
            hour_index = i % uploads_per_day

            upload_time = (current_date + timedelta(days=day_offset)).replace(
                hour=optimal_hours[hour_index],
                minute=0
            )

            scheduled.append(ScheduledUpload(
                short=short,
                channel_id=channel_id,
                scheduled_for=upload_time
            ))

        return scheduled
```

## Shorts sem Checkpoint Humano

Shorts extraídos de long-form **não passam por checkpoint humano** porque:

1. O conteúdo já foi aprovado no CHECKPOINT 1 (script)
2. O audio já foi aprovado no CHECKPOINT 2
3. O B-roll já foi aprovado no CHECKPOINT 3
4. A extração é automática e determinística

**Exceção:** Shorts originais (standalone) podem opcionalmente passar por checkpoint se configurado.

## Métricas de Shorts

| Métrica | Meta |
|---------|------|
| Produção | 200/mês por canal |
| Shorts por long-form | 3-5 extraídos |
| Duração média | 30-45 segundos |
| Upload success rate | >99% |
| Avg views (30 dias) | Depende do nicho |

---

**Anterior:** [04_LONGFORM_PIPELINE.md](./04_LONGFORM_PIPELINE.md)
**Próximo:** [06_THUMBNAIL_SYSTEM.md](./06_THUMBNAIL_SYSTEM.md)

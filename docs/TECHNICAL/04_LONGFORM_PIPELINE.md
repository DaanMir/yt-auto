# 04 - Long-form Video Pipeline

## Visão Geral

Pipeline completo para produção de vídeos longos (8-15 minutos). Recebe um script aprovado e produz um vídeo final pronto para upload no YouTube.

**Input:** Script aprovado (CHECKPOINT 1)
**Output:** Vídeo MP4 + Thumbnail + Metadata

## Pipeline Completo (16 Steps)

```
STEP 1:  Topic submission (User)
STEP 2:  Workflow creation (Orchestrator)
STEP 3:  Niche routing (Niche Router)
STEP 4:  Knowledge query (Knowledge System)
STEP 5:  Research (Research Pipeline → doc 03)
STEP 6:  ★ CHECKPOINT 1: Script review
STEP 7:  Audio generation (Audio Pipeline)
STEP 8:  ★ CHECKPOINT 2: Audio review
STEP 9:  B-roll search & download (B-roll Pipeline)
STEP 10: ★ CHECKPOINT 3: B-roll review
STEP 11: Video assembly (Assembly Pipeline)
STEP 12: Thumbnail generation (Thumbnail Pipeline → doc 06)
STEP 13: ★ CHECKPOINT 4: Final review
STEP 14: YouTube upload (Upload System)
STEP 15: Knowledge extraction (Knowledge System)
STEP 16: Analytics tracking
```

## Audio Pipeline (Step 7)

### Azure TTS - Free Tier

```python
class AudioPipeline:
    def __init__(self):
        self.speech_config = speechsdk.SpeechConfig(
            subscription=os.getenv("AZURE_TTS_KEY"),
            region=os.getenv("AZURE_TTS_REGION")
        )

    def generate_audio(self, script: Script, niche_config: dict) -> AudioResult:
        """Gera narração a partir do script."""

        audio_config = niche_config["audio_config"]

        # Configurar voz
        self.speech_config.speech_synthesis_voice_name = audio_config["voice_id"]

        # Preparar SSML para controle fino
        ssml = self._build_ssml(
            text=script.text,
            speed=audio_config["speed"],
            pitch=audio_config["pitch"],
            style=audio_config["style"]
        )

        # Gerar audio
        synthesizer = speechsdk.SpeechSynthesizer(
            speech_config=self.speech_config,
            audio_config=speechsdk.audio.AudioOutputConfig(
                filename=f"output/audio/{script.topic_slug}.wav"
            )
        )

        result = synthesizer.speak_ssml_async(ssml).get()

        if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
            return AudioResult(
                file_path=f"output/audio/{script.topic_slug}.wav",
                duration_seconds=self._get_duration(result),
                voice=audio_config["voice_id"],
                success=True
            )
        else:
            raise AudioGenerationError(f"TTS failed: {result.reason}")

    def _build_ssml(self, text: str, speed: float, pitch: int, style: str) -> str:
        """Constrói SSML com pausas naturais e ênfases."""

        # Inserir pausas em pontuação
        text = text.replace(". ", '. <break time="400ms"/> ')
        text = text.replace("? ", '? <break time="500ms"/> ')
        text = text.replace("! ", '! <break time="300ms"/> ')

        return f"""
        <speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis"
               xmlns:mstts="https://www.w3.org/2001/mstts" xml:lang="pt-BR">
            <voice name="{self.speech_config.speech_synthesis_voice_name}">
                <mstts:express-as style="{style}">
                    <prosody rate="{speed}" pitch="{pitch:+d}Hz">
                        {text}
                    </prosody>
                </mstts:express-as>
            </voice>
        </speak>
        """
```

### Limites Free Tier Azure TTS

| Recurso | Limite | Suficiente para |
|---------|--------|----------------|
| Caracteres/mês | 500.000 | ~80 vídeos de 10 min |
| Requisições/segundo | 20 | Sem problema |
| Vozes disponíveis | Todas neural | OK |

## B-roll Pipeline (Step 9)

### Busca em Pexels + Pixabay

```python
class BrollPipeline:
    def __init__(self):
        self.pexels = PexelsClient(os.getenv("PEXELS_API_KEY"))
        self.pixabay = PixabayClient(os.getenv("PIXABAY_API_KEY"))

    def find_broll(self, script: Script, audio_duration: float) -> list[BrollClip]:
        """Encontra B-roll relevante para cada segmento do script."""

        # 1. Extrair visual markers do script
        markers = self._extract_visual_markers(script.text)

        # 2. Para cada marker, buscar clips
        clips = []
        for marker in markers:
            search_results = self._search_clips(marker.description)

            if not search_results:
                # Fallback: query mais genérica
                search_results = self._search_clips(
                    self._generalize_query(marker.description)
                )

            best_clip = self._select_best_clip(search_results, marker)
            clips.append(best_clip)

        # 3. Preencher gaps (segmentos sem visual marker)
        clips = self._fill_gaps(clips, script, audio_duration)

        # 4. Download
        for clip in clips:
            clip.local_path = self._download_clip(clip)

        return clips

    def _search_clips(self, query: str) -> list[SearchResult]:
        """Busca em Pexels e Pixabay simultaneamente."""

        pexels_results = self.pexels.search_videos(
            query=query,
            orientation="landscape",
            size="medium",
            per_page=10
        )

        pixabay_results = self.pixabay.search_videos(
            q=query,
            video_type="film",
            per_page=10
        )

        # Merge e rank por relevância
        all_results = pexels_results + pixabay_results
        return sorted(all_results, key=lambda x: x.relevance_score, reverse=True)

    def _fill_gaps(self, clips, script, audio_duration):
        """Garante cobertura visual completa do vídeo."""
        total_clip_duration = sum(c.duration for c in clips)
        gap = audio_duration - total_clip_duration

        if gap > 0:
            # Adicionar clips genéricos do nicho
            filler_clips = self._get_niche_filler_clips(
                script.niche_id, gap
            )
            clips.extend(filler_clips)

        return clips
```

## Assembly Pipeline (Step 11)

### Composição do Vídeo Final

```python
class AssemblyPipeline:
    def assemble(self, audio: AudioResult, clips: list[BrollClip],
                  script: Script, niche_config: dict) -> VideoResult:
        """Monta o vídeo final combinando audio + B-roll + subtitles."""

        video_config = niche_config["video_config"]

        # 1. Gerar legendas com Whisper
        subtitles = self._generate_subtitles(audio.file_path)

        # 2. Preparar timeline
        timeline = self._create_timeline(audio, clips, subtitles)

        # 3. Renderizar com MoviePy
        video = self._render_video(timeline, video_config)

        # 4. Adicionar intro/outro (se configurado)
        video = self._add_intro_outro(video, niche_config)

        # 5. Export final
        output_path = f"output/videos/{script.topic_slug}_final.mp4"
        video.write_videofile(
            output_path,
            fps=video_config["fps"],
            codec="libx264",
            audio_codec="aac",
            bitrate="5000k",
            preset="medium"
        )

        return VideoResult(
            file_path=output_path,
            duration=video.duration,
            resolution=video_config["resolution"],
            file_size=os.path.getsize(output_path)
        )

    def _generate_subtitles(self, audio_path: str) -> list[Subtitle]:
        """Gera legendas usando Whisper (local, gratuito)."""
        import whisper

        model = whisper.load_model("base")
        result = model.transcribe(audio_path, language="pt")

        subtitles = []
        for segment in result["segments"]:
            subtitles.append(Subtitle(
                start=segment["start"],
                end=segment["end"],
                text=segment["text"].strip()
            ))

        return subtitles

    def _create_timeline(self, audio, clips, subtitles):
        """Cria timeline sincronizada."""

        timeline = Timeline(duration=audio.duration)

        # Layer 1: Audio
        timeline.add_audio(audio.file_path, start=0)

        # Layer 2: B-roll clips (sequencial, cortados para caber)
        current_time = 0
        for clip in clips:
            clip_duration = min(clip.duration, audio.duration - current_time)
            timeline.add_video(
                clip.local_path,
                start=current_time,
                duration=clip_duration
            )
            current_time += clip_duration

        # Layer 3: Subtitles overlay
        for sub in subtitles:
            timeline.add_subtitle(sub.text, sub.start, sub.end)

        return timeline
```

### FFmpeg Settings

```python
FFMPEG_SETTINGS = {
    "longform": {
        "resolution": "1920x1080",
        "fps": 30,
        "video_codec": "libx264",
        "video_bitrate": "5000k",
        "audio_codec": "aac",
        "audio_bitrate": "192k",
        "preset": "medium",  # balance speed vs quality
        "crf": 23,           # constant rate factor
        "pixel_format": "yuv420p"
    }
}
```

## Upload System (Step 14)

```python
class UploadSystem:
    def __init__(self):
        self.youtube = self._authenticate_youtube()

    def upload(self, video: VideoResult, thumbnail_path: str,
                metadata: dict, niche_config: dict) -> UploadResult:
        """Upload para YouTube com metadata otimizada."""

        upload_config = niche_config["upload_config"]

        # Calcular melhor horário de publicação
        schedule_time = self._calculate_schedule(upload_config)

        body = {
            "snippet": {
                "title": metadata["title"],
                "description": metadata["description"],
                "tags": metadata["tags"] + upload_config["default_tags"],
                "categoryId": upload_config["category_id"],
                "defaultLanguage": "pt"
            },
            "status": {
                "privacyStatus": upload_config["privacy"],
                "publishAt": schedule_time.isoformat() if schedule_time else None,
                "selfDeclaredMadeForKids": False
            }
        }

        # Upload video
        request = self.youtube.videos().insert(
            part="snippet,status",
            body=body,
            media_body=MediaFileUpload(video.file_path, resumable=True)
        )
        response = self._resumable_upload(request)

        # Upload thumbnail
        self.youtube.thumbnails().set(
            videoId=response["id"],
            media_body=MediaFileUpload(thumbnail_path)
        ).execute()

        return UploadResult(
            video_id=response["id"],
            url=f"https://youtube.com/watch?v={response['id']}",
            scheduled_for=schedule_time,
            channel_id=metadata["channel_id"]
        )
```

## Estrutura de Output

```
output/
├── {channel_id}/
│   ├── {video_slug}/
│   │   ├── script.md           # Script aprovado
│   │   ├── audio.wav           # Narração
│   │   ├── subtitles.srt       # Legendas
│   │   ├── broll/              # Clips de B-roll
│   │   │   ├── clip_001.mp4
│   │   │   ├── clip_002.mp4
│   │   │   └── ...
│   │   ├── thumbnail.png       # Thumbnail final
│   │   ├── video_final.mp4     # Vídeo final
│   │   └── metadata.json       # Metadata + analytics
│   └── ...
└── ...
```

## Métricas do Pipeline

| Métrica | Descrição | Meta |
|---------|-----------|------|
| Script to Video | Tempo total de produção | <30 min |
| Audio Quality | Score de naturalidade | >8/10 |
| B-roll Coverage | % do vídeo com visual relevante | >90% |
| File Size | Tamanho médio do vídeo final | <500 MB |
| Upload Success Rate | % de uploads sem erro | >99% |

---

**Anterior:** [03_RESEARCH_PIPELINE.md](./03_RESEARCH_PIPELINE.md)
**Próximo:** [05_SHORTS_PIPELINE.md](./05_SHORTS_PIPELINE.md)

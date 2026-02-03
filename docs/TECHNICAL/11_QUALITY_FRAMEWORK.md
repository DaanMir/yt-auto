# Quality Assurance Framework - Q-Score System

## Objetivo

Garantir qualidade profissional consistente através de validação automática + humana antes do upload.

## Q-Score Components

### 1. Variação de Tom (30%)
**Medição:** Análise de áudio via Librosa (Pitch variance)

**Threshold:** Score >= 7.0

**Ação se falhar:**
- Regenerar TTS com modelo "mais expressivo"
- Tentar voz alternativa do mesmo idioma
- Flag para review humano se falhar 2x

**Implementação:**
```python
import librosa
import numpy as np

def calculate_tone_variance(audio_path):
    y, sr = librosa.load(audio_path)
    pitches, magnitudes = librosa.piptrack(y=y, sr=sr)
    
    # Extract pitch variance
    pitch_values = []
    for t in range(pitches.shape[1]):
        index = magnitudes[:, t].argmax()
        pitch = pitches[index, t]
        if pitch > 0:
            pitch_values.append(pitch)
    
    variance = np.std(pitch_values)
    score = min(10, variance * 10)  # Normalize to 0-10
    return score
```

### 2. Pacing Visual (30%)
**Medição:** Scene change rate via FFmpeg

**Threshold:** Score >= 7.0 (optimal: 1 cut every 3-5s)

**Ação se falhar:**
- Adicionar B-roll extra
- Inserir mais transições
- Speed up B-roll clips (1.1x - 1.2x)

**Implementação:**
```python
import cv2

def calculate_pacing_score(video_path):
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    # Detect scene changes
    scene_changes = detect_scene_changes(cap)
    
    # Optimal: 1 cut every 3-5 seconds
    duration_seconds = total_frames / fps
    optimal_cuts = duration_seconds / 4  # 1 cut every 4s (middle of range)
    
    actual_cuts = len(scene_changes)
    ratio = actual_cuts / optimal_cuts
    
    # Score: 10 if perfect, degrade if too few or too many
    if 0.7 <= ratio <= 1.3:
        score = 10
    else:
        score = max(0, 10 - abs(ratio - 1) * 10)
    
    return score
```

### 3. Relevância Semântica (20%)
**Medição:** CLIP Score (embeddings Script vs B-roll)

**Threshold:** Score >= 7.0

**Ação se falhar:**
- Substituir clips com baixo similarity score
- Re-search com keywords refinados
- Flag para review humano

**Implementação:**
```python
import torch
import clip
from PIL import Image

def calculate_semantic_relevance(script_segment, video_frame_path):
    device = "cuda" if torch.cuda.is_available() else "cpu"
    model, preprocess = clip.load("ViT-B/32", device=device)
    
    # Encode text
    text = clip.tokenize([script_segment]).to(device)
    
    # Encode image
    image = preprocess(Image.open(video_frame_path)).unsqueeze(0).to(device)
    
    # Calculate similarity
    with torch.no_grad():
        text_features = model.encode_text(text)
        image_features = model.encode_image(image)
        
        similarity = torch.cosine_similarity(text_features, image_features)
    
    score = similarity.item() * 10  # Normalize to 0-10
    return score
```

### 4. Avaliação Humana (20%)
**Medição:** Checklist rápido (3 perguntas sim/não)

**Checklist:**
1. ✓ O vídeo prende atenção nos primeiros 5s? (hook effectiveness)
2. ✓ O áudio é claro e natural? (TTS quality)
3. ✓ As imagens são relevantes ao conteúdo? (B-roll alignment)

**Scoring:**
- 3/3 = 10 pontos
- 2/3 = 7 pontos
- 1/3 = 3 pontos
- 0/3 = 0 pontos (reject)

## Calculation

```python
def calculate_q_score(video_data):
    scores = {
        'tone_variance': calculate_tone_variance(video_data['audio_path']),
        'pacing_visual': calculate_pacing_score(video_data['video_path']),
        'semantic_relevance': calculate_semantic_relevance_avg(video_data),
        'human_eval': video_data['human_checklist_score']
    }
    
    weights = {
        'tone_variance': 0.30,
        'pacing_visual': 0.30,
        'semantic_relevance': 0.20,
        'human_eval': 0.20
    }
    
    q_score = sum(scores[k] * weights[k] for k in scores)
    
    return {
        'total_score': q_score,
        'breakdown': scores,
        'pass': q_score >= 7.0
    }
```

## Integration in Pipeline

### Before Upload (Checkpoint 5.5)

```
1. Video assembled → Calculate Q-Score
2. IF Q-Score < 7.0:
   - Identify failing component(s)
   - Auto-fix if possible (regenerate TTS, add B-roll)
   - Re-calculate Q-Score
3. IF still < 7.0 after auto-fix:
   - Flag for human review
   - Dashboard shows specific issues
4. IF Q-Score >= 7.0:
   - Proceed to Checkpoint 6 (Upload Approval)
```

## Dashboard Integration

**New Widget: Quality Metrics**
```
┌─────────────────────────────────┐
│ VIDEO QUALITY BREAKDOWN         │
├─────────────────────────────────┤
│ Q-Score: 8.2 ✓                  │
│                                 │
│ Tone Variance:    8.5 ✓ (30%)  │
│ Pacing Visual:    7.8 ✓ (30%)  │
│ Semantic Rel.:    8.0 ✓ (20%)  │
│ Human Eval:       9.0 ✓ (20%)  │
│                                 │
│ Status: Ready for Upload        │
└─────────────────────────────────┘
```

## Niche Validation Protocol (NVP)

### Phase 1: Sondagem (Week 1)
**Goal:** Validate audience interest with minimal investment

**Actions:**
- Produce 5 Shorts (30-60s)
- Low effort: Static images + TTS only
- No custom B-roll, use stock only

**Success Criteria:**
- Combined views: >= 1,000
- CTR: >= 5%

**Decision:**
- IF success → Proceed to Phase 2
- IF fail → Pivot niche or angle

### Phase 2: Piloto (Week 2)
**Goal:** Validate long-form content quality

**Actions:**
- Produce 3 long-form videos (5 min)
- Full quality: Research + B-roll + Q-Score validation
- Apply all checkpoints

**Success Criteria:**
- CTR: >= 4%
- AVD (Average View Duration): >= 30%
- Subscriber conversion: >= 0.5%

**Decision:**
- IF success → Proceed to Phase 3
- IF fail → Revise content strategy, retry Phase 2

### Phase 3: Scale-Up (Month 1)
**Goal:** Full production

**Actions:**
- Activate full automation (40 videos/month)
- Monitor Q-Score trends
- Track channel health metrics

**Success Criteria:**
- Maintain CTR >= 5%
- Q-Score average >= 7.5
- Subscriber growth rate positive

## Product Roadmap Integration

### Months 1-2: Automated Pacing
**Objective:** Remove manual pacing validation

**Tasks:**
- Integrate FFmpeg scene detection in pipeline
- Auto-calculate pacing score
- Auto-add B-roll if score < 7.0

**Deliverable:** Pacing score automated, human validation optional

### Months 3-4: Semantic Validation
**Objective:** Ensure B-roll relevance automatically

**Tasks:**
- Integrate CLIP model in B-roll pipeline
- Calculate similarity score per clip
- Auto-replace low-scoring clips

**Deliverable:** Semantic relevance automated

### Months 5-6: Predictive Health Dashboard
**Objective:** Prevent channel decline before it happens

**Tasks:**
- Build ML model (trend prediction)
- Track: CTR decay, AVD trends, subscriber churn
- Alert if metrics degrading

**Deliverable:** Proactive alerts ("Channel X needs content refresh")

## Monitoring & Alerts

### Critical Alerts
- Q-Score < 6.0 for 3+ consecutive videos
- Channel CTR dropped > 20% in 7 days
- AVD < 25% (audience not engaged)

### Warning Alerts
- Q-Score trending down (last 10 videos)
- Semantic relevance consistently < 7.5
- Human checklist failing 1+ items repeatedly

---

**This framework ensures quality scales with production volume.**

# 06 - Thumbnail System

## Visão Geral

O sistema de thumbnails gera imagens atraentes para cada vídeo usando templates do Canva Pro ($13/mês). Thumbnails são gerados automaticamente e apresentados no CHECKPOINT 4 para aprovação.

**Regra fundamental:** Thumbnails funcionam melhor **sem rostos** neste sistema.

**Meta CTR:** 6-9%

## Princípios de Design

1. **Bold text** - Texto grande, legível em mobile (5 palavras máximo)
2. **High contrast** - Cores vibrantes, contraste forte
3. **No faces** - Sem rostos humanos (melhor CTR para canais sem apresentador)
4. **Niche-consistent** - Paleta visual consistente por nicho
5. **Curiosity gap** - Thumbnail deve criar curiosidade

## Arquitetura

```
SCRIPT APROVADO
   ↓
┌──────────────────────┐
│ 1. Title Analysis    │ ← LLM extrai conceito visual do título
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 2. Template Select   │ ← Seleciona template Canva do nicho
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 3. Text Generation   │ ← LLM gera texto da thumbnail (max 5 palavras)
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 4. Background Search │ ← Busca imagem de fundo (Pexels/Pixabay)
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 5. Composition       │ ← Renderiza via Canva API ou Pillow
└──────┬───────────────┘
       ↓
┌──────────────────────┐
│ 6. Variants          │ ← Gera 3 variantes para A/B test
└──────┘
       ↓
   CHECKPOINT 4: Review (junto com vídeo final)
```

## Configuração por Nicho

```json
{
  "thumbnail_config": {
    "style": "bold_text_gradient",
    "max_text_words": 5,
    "use_faces": false,
    "color_palette": ["#1a1a2e", "#16213e", "#0f3460", "#e94560"],
    "font_family": "Montserrat",
    "font_weight": "ExtraBold",
    "text_position": "center",
    "text_shadow": true,
    "background_style": "gradient_overlay",
    "icon_style": "emoji_or_icon",
    "resolution": "1280x720",
    "format": "png"
  }
}
```

## Estilos de Thumbnail

### Style: `bold_text_gradient`
- Fundo com gradiente + imagem
- Texto centralizado em bold
- Bom para: finanças, educação, tech

### Style: `split_comparison`
- Tela dividida (antes/depois, vs)
- Texto em cada lado
- Bom para: reviews, comparações

### Style: `number_highlight`
- Número grande em destaque
- Subtexto menor abaixo
- Bom para: listas, rankings, dados

### Style: `minimal_icon`
- Fundo sólido/gradiente
- Ícone/emoji centralizado
- Texto curto
- Bom para: tutoriais, how-to

## Implementação

### Text Generation

```python
class ThumbnailGenerator:
    def generate_text(self, title: str, niche_config: dict) -> ThumbnailText:
        """Gera texto impactante para thumbnail."""

        prompt = f"""
        Crie texto para thumbnail do YouTube baseado no título do vídeo.

        Título do vídeo: "{title}"

        Regras:
        - Máximo {niche_config['thumbnail_config']['max_text_words']} palavras
        - Deve criar curiosidade
        - NÃO repetir o título exatamente
        - Deve ser legível em tela pequena (mobile)
        - Use números quando possível (ex: "5X MAIS")
        - Pode usar emojis estrategicamente

        Retorne 3 opções em JSON:
        [
            {{"text": "...", "style_hint": "..."}}
        ]
        """

        response = self.groq_client.chat(prompt)
        return self._parse_options(response)
```

### Background Search

```python
def find_background(self, title: str, niche_id: str) -> str:
    """Busca imagem de fundo relevante."""

    # Query simplificada do título
    query = self._simplify_for_image_search(title)

    # Buscar em Pexels (prioridade: fotos, não vídeos)
    results = self.pexels.search_photos(
        query=query,
        orientation="landscape",
        size="large",
        per_page=5
    )

    if not results:
        # Fallback: imagem genérica do nicho
        results = self.pexels.search_photos(
            query=self.niche_defaults[niche_id]["generic_bg_query"],
            orientation="landscape"
        )

    # Selecionar melhor (mais contraste, menos busy)
    return self._select_best_background(results)
```

### Composição com Pillow

```python
from PIL import Image, ImageDraw, ImageFont, ImageFilter

def compose_thumbnail(self, text: str, bg_image_path: str,
                       config: dict) -> str:
    """Renderiza thumbnail final."""

    # 1. Preparar background
    bg = Image.open(bg_image_path).resize((1280, 720))

    # 2. Aplicar overlay (gradient ou darken)
    if config["background_style"] == "gradient_overlay":
        overlay = self._create_gradient_overlay(
            1280, 720, config["color_palette"]
        )
        bg = Image.blend(bg, overlay, alpha=0.6)
    elif config["background_style"] == "darken":
        bg = bg.point(lambda p: p * 0.4)

    draw = ImageDraw.Draw(bg)

    # 3. Adicionar texto
    font = ImageFont.truetype(
        f"fonts/{config['font_family']}-{config['font_weight']}.ttf",
        size=self._calculate_font_size(text, 1280)
    )

    # Posicionar texto
    text_bbox = draw.textbbox((0, 0), text, font=font)
    text_width = text_bbox[2] - text_bbox[0]
    text_height = text_bbox[3] - text_bbox[1]
    x = (1280 - text_width) // 2
    y = (720 - text_height) // 2

    # Shadow
    if config.get("text_shadow"):
        draw.text((x + 3, y + 3), text, font=font, fill="#000000")

    # Main text
    draw.text((x, y), text, font=font, fill="#FFFFFF")

    # 4. Salvar
    output_path = f"output/thumbnails/{uuid4()}.png"
    bg.save(output_path, "PNG", quality=95)

    return output_path

def _create_gradient_overlay(self, width, height, colors):
    """Cria overlay com gradiente usando paleta do nicho."""
    overlay = Image.new("RGB", (width, height))
    draw = ImageDraw.Draw(overlay)

    for y in range(height):
        ratio = y / height
        r = int(int(colors[0][1:3], 16) * (1 - ratio) + int(colors[-1][1:3], 16) * ratio)
        g = int(int(colors[0][3:5], 16) * (1 - ratio) + int(colors[-1][3:5], 16) * ratio)
        b = int(int(colors[0][5:7], 16) * (1 - ratio) + int(colors[-1][5:7], 16) * ratio)
        draw.line([(0, y), (width, y)], fill=(r, g, b))

    return overlay
```

### Geração de Variantes

```python
def generate_variants(self, title: str, bg_path: str,
                       config: dict) -> list[str]:
    """Gera 3 variantes para seleção humana."""

    text_options = self.generate_text(title, config)
    variants = []

    for i, option in enumerate(text_options[:3]):
        # Variar estilo levemente
        variant_config = config.copy()

        if i == 1:
            variant_config["background_style"] = "darken"
        elif i == 2:
            variant_config["text_position"] = "bottom"

        path = self.compose_thumbnail(
            option.text, bg_path, variant_config
        )
        variants.append({
            "path": path,
            "text": option.text,
            "variant": i + 1
        })

    return variants
```

## Canva Pro Integration (Alternativo)

Para templates mais sofisticados, usar Canva Pro API:

```python
class CanvaIntegration:
    def create_from_template(self, template_id: str, data: dict) -> str:
        """Cria thumbnail a partir de template Canva."""

        # 1. Clonar template
        design = self.canva_api.create_design(
            template_id=template_id,
            title=f"thumb_{data['video_slug']}"
        )

        # 2. Atualizar textos e imagens
        self.canva_api.update_elements(design.id, {
            "main_text": data["thumbnail_text"],
            "background_image": data["bg_image_url"]
        })

        # 3. Exportar como PNG
        export = self.canva_api.export(design.id, format="png")
        return self._download_export(export.url)
```

## Métricas

| Métrica | Meta |
|---------|------|
| CTR médio | 6-9% |
| Variantes por vídeo | 3 |
| Geração (tempo) | <30 segundos |
| Custo Canva Pro | $13/mês (ilimitado) |
| A/B test win rate | Track qual variante performa melhor |

---

**Anterior:** [05_SHORTS_PIPELINE.md](./05_SHORTS_PIPELINE.md)
**Próximo:** [07_DATABASE_SCHEMA.md](./07_DATABASE_SCHEMA.md)

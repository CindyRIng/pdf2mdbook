# Equal-Margin Figure Cropping

When extracting figures from PDFs, the clip-based extraction often produces asymmetric margins — the top is tight, the bottom is loose, left/right differ. This reference provides a two-phase Python pattern for producing figures with uniform tiny white margins on all four sides.

## Two-Phase Workflow

### Phase 1: Tight PDF Clip

Extract the figure region from the PDF with minimal padding:

```python
import fitz

doc = fitz.open(pdf_path)
page = doc[pdf_page_num]

# Figure content bounds in PDF points (determined from text blocks + drawings)
fig_x0, fig_y0, fig_x1, fig_y1 = 52, 361, 450, 582

tight_pad = 2  # minimal — just enough to not cut edges
crop_rect = fitz.Rect(
    max(0, fig_x0 - tight_pad),
    max(0, fig_y0 - tight_pad),
    min(page.rect.width, fig_x1 + tight_pad),
    min(page.rect.height, fig_y1 + tight_pad)
)

zoom = 4  # 288 DPI for crisp output
mat = fitz.Matrix(zoom, zoom)
pix = page.get_pixmap(matrix=mat, clip=crop_rect)
```

### Phase 2: PIL Ink Detection + Equal Padding

Convert to PIL, detect actual ink bounds, crop to content, then add uniform padding:

```python
from PIL import Image, ImageOps
import numpy as np

img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)

# Detect ink (non-white pixels)
gray = np.array(img.convert("L"))
ink = gray < 240  # threshold for non-white
rows = np.any(ink, axis=1)
cols = np.any(ink, axis=0)

ink_top = np.argmax(rows)
ink_bottom = len(rows) - np.argmax(rows[::-1]) - 1
ink_left = np.argmax(cols)
ink_right = len(cols) - np.argmax(cols[::-1]) - 1

# Crop to just the ink
ink_img = img.crop((ink_left, ink_top, ink_right + 1, ink_bottom + 1))

# Add equal border on all 4 sides
margin_px = 50  # ~12.5pt at 4× zoom — "微小留白"
final_img = ImageOps.expand(ink_img, border=margin_px, fill='white')
final_img.save(output_path)
```

## Visual Compensation for Asymmetric Content

Some figures have top-heavy content (labels, arrows at the very top edge) that makes equal pixel margins FEEL uneven. After adding equal padding, visually verify:

```python
# If top still feels tight, add extra top padding
from PIL import Image
img = Image.open(output_path)
img = ImageOps.expand(img, border=(0, 15, 0, 0), fill='white')  # +15px top only
img.save(output_path)
```

Target: top margin should be ~15-20px larger than bottom to visually balance top-heavy figures.

## Verification

After cropping, run `vision_analyze` on the output:
- "这张图上下左右的留白是否等距？顶部是否有文字或图形被切断？"
- If top still feels tight → add 15px extra top padding
- If left/right differ → check page-edge constraints

## Common Pitfall

- **Page edge limits symmetric padding.** If the figure extends to the page right edge, max symmetric horizontal padding is capped by `page.rect.width - fig_x1`. In that case, use that cap as the uniform margin on all sides, then visually compensate top if needed.

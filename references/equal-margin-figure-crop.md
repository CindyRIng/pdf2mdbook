# Equal-Margin Figure Cropping

When extracting figures from PDFs, the clip-based extraction often produces asymmetric margins — the top is tight, the bottom is loose, left/right differ. This reference provides a complete Python pattern for producing figures with uniform white margins on all four sides.

## Complete Pipeline

### Step 1: Locate Figure Pages (Text Search)

```python
import fitz
doc = fitz.open(pdf_path)
for i in range(len(doc)):
    if 'Figure 3.2' in doc[i].get_text():
        print(f'Figure referenced on PDF page {i+1}')
```

The figure may be on the referenced page, the next page, or a nearby page — always render candidates around the reference.

### Step 2: Render Candidates + Vision Locate

```python
mat = fitz.Matrix(300/72, 300/72)
for pg in candidate_pages:
    page = doc[pg]
    pix = page.get_pixmap(matrix=mat)
    pix.save(f'/tmp/page_{pg+1}.png')
```

Then use `vision_analyze` on each candidate:
> "Where exactly is Figure X.Y on this page? Give top%, bottom%, left%, right%."

### Step 3: Refine with Text Block Analysis

Vision proportions are approximate — they can't precisely distinguish a caption's bottom edge from a nearby horizontal page rule. Use pymupdf `get_text("blocks")` for pixel-perfect boundaries:

```python
page = doc[pg_idx]
for b in page.get_text("blocks"):
    x0, y0, x1, y1 = b[:4]
    text = b[4].strip()[:80]
    pct_y = y0 / page.rect.height
    pct_x = x0 / page.rect.width
    if any(kw in text for kw in ['Figure', 'Caption', 'Context', 'Subdomain']):
        print(f'  Block: y={y0:.0f}-{y1:.0f} ({pct_y*100:.0f}%-{y1/page.rect.height*100:.0f}%) x={x0:.0f}-{x1:.0f} text={text!r}')
```

This gives you exact Y coordinates for:
- Where body text above the figure ends
- Where figure labels sit
- Where the caption starts and ends
- Where any page separator rules appear

### Step 4: Set Crop Boundaries

Combine vision proportions with text block data, set `v_bot` BETWEEN the caption's bottom and any page rule:

```python
w, h = page.rect.width, page.rect.height
mat = fitz.Matrix(300/72, 300/72)

crop_rect = fitz.Rect(
    v_left * w, v_top * h,
    v_right * w, v_bot * h
)
pix = page.get_pixmap(matrix=mat, clip=crop_rect)
```

### Step 5: Ink Detection + Equal Padding

```python
from PIL import Image, ImageOps
import numpy as np

img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)

# Detect ink (non-white pixels)
gray = np.array(img.convert("L"))
ink = gray < 245  # threshold for near-white
rows = np.any(ink, axis=1)
cols = np.any(ink, axis=0)

ink_top = int(rows.argmax())
ink_bot = int(len(rows) - rows[::-1].argmax() - 1)
ink_left = int(cols.argmax())
ink_right = int(len(cols) - cols[::-1].argmax() - 1)

# Crop to just the ink
ink_img = img.crop((ink_left, ink_top, ink_right + 1, ink_bot + 1))

# Add equal border on all 4 sides
margin_px = 50  # ~12.5pt at 4× zoom — "微小留白"
final_img = ImageOps.expand(ink_img, border=margin_px, fill='white')
final_img.save(output_path)
```

### Step 6: Exclude Horizontal Separator Rules

Many book pages have a gray horizontal rule between the figure caption and the body text below. Ink detection (Step 5) will capture this rule as content. To exclude it:

**Detection** — scan for rows where most pixels are dark:
```python
dark_ratio = (gray < 200).mean(axis=1)
rule_candidates = [(i, dark_ratio[i]) for i in range(len(dark_ratio)) if dark_ratio[i] > 0.7]
```

A row with >70% dark pixels spanning the full width is a horizontal rule, not figure content.

**Solution** — set `v_bot` ABOVE the rule, using text block coordinates to find the precise gap between caption end and rule start. In practice, this can be as tight as 0.002-0.005 of page height:

```
Caption ends at y=441 → 0.679 of page height
Rule starts at    y=460 → 0.708 of page height
Target v_bot     = 0.685  (between them)
```

### Step 7: Visual Compensation

If the figure has top-heavy elements (labels, arrows at the very top edge), equal pixel margins may feel unbalanced. After adding equal padding:

```python
# If top still feels tight, add extra top padding
final_img = ImageOps.expand(final_img, border=(15, 0, 0, 0), fill='white')
```

Similarly, if the caption sits close to the figure's bottom, the bottom may feel tight relative to the top:

```python
final_img = ImageOps.expand(final_img, border=(0, 0, 0, 10), fill='white')
```

Target: visually balanced margins where no side feels cramped.

### Step 8: Vision Verify

Run `vision_analyze` on the output after EVERY round:
> "Is this figure cropped correctly? ONLY the figure + complete caption, with visually balanced equal white margins on all four sides? No body text? No horizontal rule?"

Expect 2-4 refinement rounds per image. The vision tool can catch what the code can't: visual imbalance, stray text fragments, and content that "feels" off.

## Common Pitfalls

- **Page edge limits symmetric padding.** If the figure extends to the page edge, max symmetric padding is capped by the page boundary. Use that cap as the uniform margin.
- **Vision proportions are approximate.** A vision estimate of "42-72%" may actually need 44.5-68.5% after inspecting text blocks. Always cross-reference with `get_text("blocks")`.
- **Horizontal rules are stealth content.** They look like figure dividers to human eyes but are page decoration. Ink detection treats them as content — explicitly exclude them.
- **Caption bottom vs rule start** can be as little as 2-3% of page height apart. Text block analysis is the only reliable way to find this gap.
- **Visual balance ≠ mathematical equality.** A figure with a top-heavy blob diagram will look top-heavy even with equal pixel margins. Apply asymmetric compensation (Step 7), then vision-verify.

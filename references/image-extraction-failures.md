# Image Extraction Failure Modes and Fixes

Session: IMPLDDD Ch1 image repair (2026-06-05) + Ch2 complete re-extraction (2026-06-08)
Context: All figure files in FIG/ had quality problems. The root discovery on 2026-06-08 was that ALL existing FIG files from automatic extraction were wrong.

## CRITICAL: Decorative vs Real Figure Detection (Mode 7)

**Symptom:** FIG directory contains many files with plausible names (FIG-2-1.jpeg, FIG-2-2.jpeg, etc.) at 8-17KB and 200×145–347×145 pixels. They look like valid extractions but are NOT real figures.

**Root cause:** `pymupdf.get_images(full=True)` extracts ALL embedded raster images, including:
- Chapter-header decorative elements (300×119, repeated at every chapter boundary)
- Inline icons (200×145, 212×164, 163×158, 213×219)
- Small vector-graphic metadata tiles (347×145)

These are NOT the book's actual diagrams. The REAL figures (architecture diagrams, UML, flow charts) are drawn as VECTOR GRAPHICS — lines, arrows, shapes, text — which `get_images()` CANNOT extract.

**Detection heuristic:** If an extracted figure file is under 20KB or under 400×400 pixels, it is almost certainly decorative, not a real figure. Real book diagrams clip-extracted at 200 DPI are typically 50-200KB and 500×800+ pixels.

**Fix workflow for each chapter:**
1. Render all pages in the chapter range at 150 DPI
2. Use vision_analyze on each page to identify pages that contain actual figures
3. Locate exact bounding box for each figure (top%/bottom%/left%/right%)
4. Clip-extract at 200 DPI: `page.get_pixmap(clip=rect, dpi=200).save("FIG/figure.png")`
5. Verify clipped output against original page
6. Delete old decorative .jpeg files
7. Add `![[../FIG/...]]` to correct PAGE file at the figure's actual printed page number
8. Use 3 parallel subagents for efficiency (7+ pages per batch)

**Chapter 2 verified figure mapping (IMPLDDD):**

| FIG | Printed Page | PDF Page | Crop (L,T,R,B) | Size |
|-----|-------------|----------|-----------------|------|
| FIG-2-1 | 45 | 87 | 0.05, 0.08, 0.98, 0.58 | 157K |
| FIG-2-2 | 50 | 92 | 0.05, 0.52, 0.95, 0.95 | 104K |
| FIG-2-3 | 53 | 95 | 0.17, 0.67, 0.83, 0.97 | 60K |
| FIG-2-4 | 58 | 100 | 0.08, 0.08, 0.92, 0.47 | 96K |
| FIG-2-5 | 63 | 105 | 0.05, 0.68, 0.95, 0.97 | 51K |
| FIG-2-6 | 66 | 108 | 0.08, 0.08, 0.92, 0.27 | 32K |
| FIG-2-7 | 73 | 115 | 0.08, 0.05, 0.92, 0.57 | 99K |
| FIG-2-8 | 74 | 116 | 0.03, 0.42, 0.97, 0.97 | 128K |
| FIG-2-9 | 81 | 123 | 0.05, 0.03, 0.95, 0.42 | 88K |
| FIG-2-10 | 83 | 125 | 0.05, 0.04, 0.95, 0.42 | 89K |

## Failure Mode Catalog

### 1. Text Bleeding (FIG-0.5-6)
**Symptom:** Figure includes 3 paragraphs of body text from surrounding page content. 
**Root cause:** Crop region too wide — included running text before the figure.
**Fix:** Use vision_analyze to locate exact figure boundaries, tighten left/right crop inward by 2-5%.
**Vision question:** "Does this image contain body text from the book page that is NOT part of the figure?"

### 2. Over-Crop / Figure Truncation (FIG-0.5-2, FIG-0.5-3, FIG-0.5-4)
**Symptom:** Figure edges cut off — oval top missing, hexagon top cropped, "Accounts Context" oval cut.
**Root cause:** Crop boundaries set too tight based on visual guess, not measured proportions.
**Fix:** Extend crop outward by 5-8%, iterate until vision confirms completeness.
**Vision question:** "Is this figure complete? Are all shapes fully visible at all edges? Anything cropped?"

### 3. Blank Extraction (FIG-0.5-8)
**Symptom:** Output image is 100% white — no content whatsoever.
**Root cause:** Wrong page mapping or clip region pointed to whitespace area of the page.
**Fix:** Re-locate figure with vision_analyze on the full page.
**Vision question:** "Is this image completely blank? Any content at all?"

### 4. Caption Present but No Diagram (FIG-0.5-7)
**Symptom:** Shows "Figure G.7" caption and Cowboy Logic text, but NO module diagram.
**Root cause:** Clip region captured the wrong area — the figure caption was in frame but the diagram was elsewhere.
**Fix:** Re-locate the actual diagram position. May need to search adjacent pages.
**Vision question:** "What figures are on this page? Give proportions for each figure separately."

### 5. Missing Figures Entirely (All Ch1 figures)
**Symptom:** Pages have embedded images in the PDF but no FIG files exist.
**Root cause:** Image extraction step was never performed for those pages.
**Fix:** Check each page with `page.get_images(full=True)`, render and vision-locate figures, clip-extract.

### 7. Bottom Boundary Too Tight — Caption Cut Off (FIG-2-4, FIG-2-9, FIG-2-10)

**Symptom:** Diagram looks complete but the figure caption ("Figure 2.X ...") is missing or partially visible. Bottom of the diagram's bounding oval is clipped.

**Root cause:** Vision-reported bottom percentage marks the END of the diagram content, but captions sit BELOW the diagram. Using the diagram's bottom edge as the clip bottom cuts off the caption. Additionally, vision may slightly underestimate the bottom boundary.

**Fix:** Always expand the bottom boundary by **8-13%** beyond the vision-reported bottom percentage to include the caption line. Example: if vision says bottom=42%, clip at bottom=55%. Verify the clipped output includes the full caption text.

**Vision verification question:** "Is the full figure caption visible at the bottom? Does it read 'Figure 2.X ...' completely? Is the bottom edge of the diagram's oval boundary fully closed?"

**Observed:** FIG-2-4 (bottom 47%→55%), FIG-2-9 (bottom 42%→55%), FIG-2-10 (bottom 42%→55%) all required bottom expansion. Each went from ~60K to ~115K after the fix.
**Symptom:** pymupdf extracts all embedded images, but some are PDF internal tiles or decorative elements.
**Root cause:** PDF may embed images as multiple small tiles stitched together by the renderer.
**Fix:** For tile-stitched images, use clip-based extraction instead of extracting embedded images.

## IMPLDDD Full Book Figure Inventory (2026-06-08 reconstruction)

After full reconstruction: 84 PNG files across 14+appendix chapters. **0 chapters had 0 decorative JPEGs remaining at end.** Key: Ch1 has NO numbered figures (tables + Cowboy Logic only). Ch7, Ch9, Ch11, Ch12 have 0 figures.

**Chapter 10 gap:** Figures 10.1–10.6 were on pages before the scanned range (p369+). Only 10.7 (p371) and 10.8 (p376) were captured in this session — earlier pages were rendered in a prior session. 10.1–10.6 need separate extraction when those pages are available.

**Figure 14.1 gap:** Located on page ~512, which fell just outside the rendered range (started at p513). Needs extraction in a follow-up.

**ChA mapping:** 17 figures spread across printed pages ~539–575. Bounding boxes were generous estimates (0.05–0.55 height clip); verify 2-3 representative figures per book pattern.

### Chapter-by-chapter counts

| Ch | Figures | Notes |
|----|---------|-------|
| 1 | 0 | Tables 1.1-1.4, Cowboy Logic, no Figure 1.X |
| 2 | 10 | Full verified (see above) |
| 3 | 10 | 3.1-3.10 Context Maps |
| 4 | 11 | 4.1-4.11 Architecture |
| 5 | 10 | 5.1-5.10 Entities/UML |
| 6 | 3 | 6.1-6.3 Value Objects |
| 7 | 0 | Prose only |
| 8 | 4 | 8.1-8.4 Domain Events |
| 9 | 0 | Modules, Table 9.1 |
| 10 | 2+ | 10.7-10.8 captured; 10.1-10.6 pending |
| 11 | 0 | Factories |
| 12 | 0 | Repositories |
| 13 | 1 | 13.1 Integration |
| 14 | 3+ | 14.2-14.4 captured; 14.1 pending |
| A | 17 | A.1-A.17 Aggregate+Event Sourcing |
| **Total** | **71+** | ~8 figures pending (10.1-10.6 + 14.1)

For each figure:

1. **Render** full page: `page.get_pixmap(dpi=150).save("/tmp/page.png")`
2. **Locate** with vision: `vision_analyze(question="Where is Figure X.Y? Give top/bottom/left/right proportions as % of page dimensions.")`
3. **Clip** using proportions: `clip = fitz.Rect(w*left, h*top, w*right, h*bottom)`
4. **Extract**: `page.get_pixmap(clip=clip, dpi=200).save("FIG/...")`
5. **Verify** clipped output: `vision_analyze(question="Is this figure complete? Any body text? Any cropping issues?")`
6. If verification fails, adjust proportions and repeat steps 3-5
7. Add `![[../FIG/...]]` to the correct PAGE file
8. Delete old decorative .jpeg files for that chapter

## ⚠️ Parallel Vision Scanning Pipeline (Proven Pattern)

For chapters with many pages (40+), scanning every page one at a time with vision_analyze is too slow (~15-20s per page = 10-15 minutes per chapter).

**Efficient pipeline (proven on IMPLDDD Ch2, 44 pages → ~6 minutes):**

1. **Batch-render** all pages in the chapter: `for p in range(start, end+1): page.get_pixmap(dpi=150).save(f"/tmp/ch{p}.png")`
2. **Parallel delegate** — split pages across 3-4 subagents:
   - Each subagent gets ~10-15 pages and instructions: "Scan each page for numbered figures with captions like 'Figure X.Y'. Report: figure number, page number, bounding box %."
   - Toolsets: `["vision","terminal"]`
3. **Aggregate** results from all subagents into a figure→page→bounding-box map
4. **Batch clip-extract** all figures in one Python script using the map
5. **Verify** 2-3 representative figures per chapter (not all — spot-check pattern)

**Subagent instructions template:**
```
Goal: Scan rendered PDF pages at /tmp/ch2_p043.png through /tmp/ch2_p086.png. 
Find EVERY numbered figure with a caption like "Figure X.Y". 
For each: provide page number, exact bounding box (top%, bottom%, left%, right%), and caption text.
Report "NO FIGURE" for pages without any.
```

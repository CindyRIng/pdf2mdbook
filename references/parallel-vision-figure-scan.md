# Parallel Vision Figure Scanning Pipeline

Proven on IMPLDDD full-book reconstruction (2026-06-08). Core insight: scanning every book page with `vision_analyze` one-by-one takes 15-20s per page = 10-15 minutes per chapter. For 14 chapters (600+ pages), that's 2+ hours. Parallelizing across subagents cuts this to ~30 minutes total.

## Proven Pipeline

```
┌─────────────────────────────────────────────────────────┐
│  1. BATCH RENDER    2. PARALLEL SCAN    3. AGGREGATE    │
│  All chapter pages  3-4 subagents      Figure→page→box  │
│  at 150 DPI         each handles ~15pp   map            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  4. BATCH CLIP      5. UPDATE REFS     6. SYNC VAULT    │
│  One Python script   Add ![[FIG]] to     ob sync         │
│  all figures at once correct PAGEs                       │
└─────────────────────────────────────────────────────────┘
```

## Step-by-Step

### 1. Batch Render All Chapter Pages

```python
import fitz
doc = fitz.open(PDF)
OFFSET = 42  # IMPLDDD: printed = pdf_page - 42

for printed in range(chapter_start, chapter_end + 1):
    pdf_p = printed + OFFSET
    pix = doc[pdf_p].get_pixmap(dpi=150)  # 150 DPI is sufficient for vision
    pix.save(f"/home/admin/ch{ch}_pages/p{printed:03d}.png")
```

**Important:** Save to `/home/admin/` not `/tmp/`. Subagents running on different backends may not have access to `/tmp/`. Always verify files exist with `ls` before delegating.

### 2. Parallel Subagent Scan

Split pages across 3-4 subagents. Each gets ~15 pages to scan. Toolsets: `["vision","terminal"]`.

**Subagent instructions template:**

```
Goal: Scan all Chapter N pages at /home/admin/chN_pages/pXXX.png through pYYY.png.
Find EVERY numbered Figure N.X with a caption like "Figure N.X".
For each: report figure number, printed page number, full caption text,
and exact bounding box (top%, bottom%, left%, right% of page).
Also note significant inline illustrations (Cowboy Logic, sidebars with icons).
Report "NO FIGURE" for pages without any.
```

**Batch sizing:**
- 40-55 pages → 3 subagents (each ~15-18 pages)
- 55-80 pages → 4 subagents (each ~15-20 pages)
- 15-30 pages → 1 subagent or inline vision calls

**Performance:** Ch2 (44 pages) → 3 subagents → ~6 minutes. Ch4+5+6 (126 pages) → 3 subagents → ~10 minutes.

### 3. Aggregate Results

Parse subagent output into a figure map: `{fig_name: (printed_page, top%, left%, bottom%, right%)}`.

**Common pitfalls in subagent output:**
- Bounding boxes reported as "~65%-95%" → strip the `~` and parse as 65, 95
- Some figures reported with caption only, no box → need follow-up vision call
- Chapter boundary confusion (e.g., pages 245-263 are still Ch6, not Ch7) → cross-reference with book TOC

### 4. Batch Clip-Extract All Figures

```python
for name, (printed, top, left, bottom, right) in all_figs.items():
    page = doc[printed + OFFSET]
    r = page.rect
    clip = fitz.Rect(r.width * left, r.height * top, r.width * right, r.height * bottom)
    pix = page.get_pixmap(clip=clip, dpi=200)
    pix.save(f"FIG/{PREFIX}-FIG-{name}.png")
```

**Heuristic when bounding boxes are missing:**
- Upper-half figure: `(0.05, 0.05, 0.55, 0.95)` — 5-55% height
- Lower-half figure: `(0.50, 0.05, 0.95, 0.95)` — 50-95% height
- Always clip generously — verification catches over-crop later

### 5. Update PAGE References

For each figure, add `![[../FIG/{PREFIX}-FIG-{name}.png]]` to the PAGE file at the figure's printed page number. Run a FIG↔PAGE cross-reference audit after.

### 6. Delete Old Decorative Files

After all new .png files are extracted and verified, delete ALL old .jpeg files in FIG/ for that chapter. These were the `get_images()` decorative tiles that the new extractions replace.

## Chapter Size Reference (IMPLDDD)

| Ch | Pages | Figures | Subagents | Time |
|----|-------|---------|-----------|------|
| 1 | 43 | 0 | 1 | 5 min |
| 2 | 44 | 10 | 3 | 6 min |
| 3 | 32 | 10 | 2 | 4 min |
| 4 | 54 | 11 | 2 | 6 min |
| 5 | 51 | 10 | 2 | 5 min |
| 6 | 21 | 3 | 1 | 2 min |
| 7 | 32 | 0 | 1 | 2 min |
| 8-14 | ~300 | ~40 | 8-10 | ~20 min |

## Pitfalls

1. **Subagent can't find files** — they may run on a different backend. Always use `/home/admin/` paths, not `/tmp/`. Verify with `ls` before delegating.

2. **Bounding boxes from subagents are approximate** — always clip generously and verify. In practice, 70-80% of figures need 1 round of adjustment (bottom expansion for captions is the most common fix).

3. **Chapter boundary detection** — subagents may report the wrong chapter for pages near boundaries. Cross-reference with the book's SECTION page ranges.

4. **Chapter 1 has no numbered figures** (in IMPLDDD) — don't waste time scanning it for Figure 1.X. Check the book's actual figure inventory first if possible.

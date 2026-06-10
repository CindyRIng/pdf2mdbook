---
name: mdbook-conversion
description: Use when converting PDF books to Obsidian MDBOOK format. Covers directory structure, file templates, 4-step bottom-up workflow, text extraction strategies (pymupdf/OCR/vision), image extraction with visual verification against original, and common pitfalls.
version: 1.8.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [mdbook, pdf, markdown, obsidian, conversion, book]
    related_skills: [obsidian-vault, ocr-and-documents]
---

# PDF to MDBOOK Conversion

## Overview

Convert a PDF book into an Obsidian-linked MDBOOK structure — a set of Markdown files organized as book → sections → pages, with prev/next navigation, summaries, and embedded images. The design philosophy is documented in `[[E2605272 PDF转Markdown思路]]`; this skill provides the executable workflow with technical implementation details.

**Core principle:** Work bottom-up. Pages first (raw content), then leaf sections (summary + page links), then parent sections (summary + sub-section links), then the book file (summary + TOC).

## When to Use

- User says "把这个PDF转成MDBOOK" or "转换成MDBOOK格式"
- User references an MDBOOK project (e.g., `MDBOOK260503-IMPLDDD`)
- Trigger words: MDBOOK, PDF转Markdown, 书籍转换, book conversion

Don't use for:
- Single-page PDFs or short documents (overkill)
- PDFs that are just scanned images without structure (needs OCR preprocessing first)
- PDFs you haven't first inspected with at least `pymupdf` to check extractability

## Directory & File Structure

```
MDBOOK<YYMMDD>-<BOOKNAME>/
├── MDBOOK<YYMMDD>-<BOOKNAME>.md          # 书级文件
├── FIG/                                   # 图片目录
│   └── MDBOOK<YYMMDD>-<BOOKNAME>-FIG-<章节号>-<序号>.png
└── CONTENT/                               # 内容目录
    ├── MDBOOK<YYMMDD>-<BOOKNAME>-SECTION<章节号>.md
    └── MDBOOK<YYMMDD>-<BOOKNAME>-PAGE<页码>.md
```

## Templates

### Book File

````
# 文件名：MDBOOK<YYMMDD>-<BOOKNAME>.md

# 基本信息

```bookinfo
书籍全称:<书名>
作者:<作者>
出版社:<出版社>
ISBN-13:<ISBN-13>
ISBN-10:<ISBN-10>
```

# 摘要

<本书核心内容的简要概述>

# 目录

- ***<章节号>-<章节名>*** - [[<SECTION文件>]]-[[<起始页文件>]]
- ...

⚠️ **Every TOC entry MUST have both links.** The `-[[PAGE起始页]]` after `-[[SECTION]]` is required, not optional. Example: `- ***1.1-我能做DDD吗？*** - [[SECTION1.1]]-[[PAGE2]]`. Entries without `-[[PAGE...]]` break the TOC's ability to navigate to content.
````

### Section File (non-leaf — links to sub-sections AND all underlying pages)

````
# 文件名：CONTENT/MDBOOK<YYMMDD>-<BOOKNAME>-SECTION<章节号>.md

# 基本信息

```
章节全称:<章节名>
页码范围:<起始页> - <结束页>
```

# 链接

- **目录：** [[<书级文件>]]
- **上一章：** [[<上一章文件>]] | 无
- **下一章：** [[<下一章文件>]] | 无

# 摘要

<本章核心内容的简要概述>

# 子章节

- [[<子章节文件>]]
- ...

# 页面

- [[<PAGE起始页>]]
- [[<PAGE起始页+1>]]
- ...
````

⚠️ **Non-leaf SECTION `# 页面` must list ALL underlying PAGE files, not `<待补充>` and NOT sub-section links.** For example, SECTION2 (pages 43-86) must have 44 `- [[PAGE{N}]]` links from PAGE43 through PAGE86. This provides a flat page index for the entire chapter, regardless of how many sub-section levels sit between. The `# 子章节` section provides hierarchical navigation; `# 页面` provides direct page-level access.

### Section File (leaf — links directly to pages)

````
# 文件名：CONTENT/MDBOOK<YYMMDD>-<BOOKNAME>-SECTION<章节号>.md

# 基本信息

```
章节全称:<章节名>
页码范围:<起始页> - <结束页>
```

# 链接

- **目录：** [[<书级文件>]]
- **上一章：** [[<上一章文件>]] | 无
- **下一章：** [[<下一章文件>]] | 无

# 摘要

<本章核心内容的简要概述>

# 页面

- [[<页面文件>]]
- ...
````

### Page File

```
# 文件名：CONTENT/MDBOOK<YYMMDD>-<BOOKNAME>-PAGE<页码>.md

# 链接

- **目录：** [[<书级文件>]]
- **上一页：** [[<上一页文件>]] | 无
- **下一页：** [[<下一页文件>]] | 无

# 内容

## <标题(本页实际所属的章节/子节标题)>
<原书正文内容，保留原始段落结构>
![[../FIG/MDBOOK<YYMMDD>-<BOOKNAME>-FIG-<章节号>-<序号>.png]]
```

**Content cleaning rules — MUST strip before writing:**
1. **Bare page numbers** — the printed digit on its own line (e.g., `43`) is a printer's mark, not book content. Delete it.
2. **Running headers** — repeated chapter names at page top/bottom (e.g., `Chapter 2 DOMAINS, SUBDOMAINS...`) are navigation aids. Delete.
3. **Book title/author footers** — repeated on every page. Delete.
4. **"(接上页)" markers** — NEVER use `(接上页)...X` as a cross-page continuity marker. It's a translator-inserted artifact that makes reading disjointed and unnatural. Real books don't have "(continued from previous page)" — sentences flow across page breaks. Instead, take the last meaningful phrase (8–15 characters) from the previous page's content ending and prepend it to the current page's opening to create a natural sentence flow. Example: PAGE XIX ends `尽管我看不清她们的眼睛。` and PAGE XX opens `(接上页)...交流。` → rewrite as `尽管我看不清她们的眼睛，也无法和她们交流。如果我朝着飞机窗外大喊...`

**Heading selection rules:**
- If this page begins a new chapter → `## 第X章 章节名`
- If this page begins a new sub-section → `## 子节名`
- If mid-flow continuation → reuse previous page's heading or no heading
- NEVER use a heading from a different section (e.g., `## 全局概览` on a page deep into "为什么战略设计如此至关重要")

Note: pages have **no summary**. The page is the content endpoint — readers arrive here to read the original text.

## Prerequisites

Before starting any MDBOOK conversion, load the `obsidian-vault` skill and sync the vault. The MDBOOK lives inside the Obsidian vault at `Resource/MDBOOK/`; all file writes must follow the vault's sync-before-edit, sync-after-edit workflow to avoid merge conflicts.

PDFs are typically stored in `Resource/PDF/` within the vault. Locate the source PDF before beginning.

## Conversion Workflow

```
┌─────────────────────────────────────────────┐
│              PDF → MDBOOK                    │
├──────────┬──────────┬──────────┬───────────┤
│ 1.初始化  │ 2.建目录  │ 3.拆页面  │ 4.补详情  │
│ 书籍骨架  │ 章节骨架  │ 内容提取  │ 自底向上  │
└──────────┴──────────┴──────────┴───────────┘
```

### Step 1: Initialize Book

- Determine today's date (YYMMDD) and generate the book short name from the title
- Create the book directory: `MDBOOK<YYMMDD>-<BOOKNAME>/` inside `Resource/MDBOOK/`
- Create subdirectories: `FIG/` and `CONTENT/`
- Create the book-level `.md` file skeleton
- Extract book info (title, author, publisher, ISBN) from the first few pages and fill the `bookinfo` block

### Step 2: Build TOC & Section Skeleton

- Extract the original table of contents from the PDF (use `pymupdf` `get_toc()`)
- Build the TOC in the book file following the `***章节号-章节名*** - [[SECTION]]-[[起始页]]` format
- **Create the FULL section hierarchy** — not just top-level chapters. Parse the TOC tree to generate all levels: top-level chapters, L2 sub-sections, L3 sub-sections, down to the deepest leaf nodes
- For non-leaf sections, fill `# 子章节` with sub-section links immediately (known from TOC)
- For all sections, leave `# 摘要` and `# 页面` as `<待补充>` — filled in Step 4 after reading actual pages
- Fill `# 链接` with prev/next at the same depth level

**Page numbering rule:** Use the book's printed page numbers, NOT PDF absolute pages. Determine the offset:
- Frontmatter pages use Roman numerals (I, II, III... XVIII) as printed in the book
- Main body pages use Arabic numerals (1, 2, 3...) as printed in the book
- The PDF page number maps to the printed number via a fixed offset (e.g., `printed = pdf_page - 1` for frontmatter, `printed = pdf_page - 43` for main body — verify per book)
- All `页码范围` in section files and page file names must use printed numbers

### Step 3: Extract Page Content

- **Each physical book page = one PAGE file.** Do NOT put page content directly into SECTION files. SECTION files are navigation nodes only — their `# 页面` section lists wiki-links to PAGE files.
- Content pipeline for EVERY page: **Extract English text from PDF → translate faithfully to Chinese → write `# 内容` in PAGE file.** All three steps are mandatory. Skipping translation (leaving English in PAGE) or inventing content from domain knowledge (instead of translating the extracted text) both produce wrong results.
- Start from the first page referenced in the TOC. For every page in the book's page range, create a PAGE file in `CONTENT/`.
- **Images:** Extract from PDF using `page.get_images(full=True)`, name by convention (`FIG/MDBOOK<YYMMDD>-<BOOKNAME>-FIG-<章节号>-<序号>.png`), embed in correct position with `![[../FIG/...]]`. The `章节号` is the parent section ID (e.g., `1.1`, `0.5`), and `序号` resets per section.
- **Tables:** Convert to Markdown tables
- **Formulas:** Express in LaTeX (`$$...$$` or `$...$`)

After all pages for a section are created, update the leaf SECTION's `# 页面` to list every PAGE file link:

### Step 4: Bottom-Up Detail Population

- Start from the **leaf** (deepest) sections
- After reading all pages under a leaf section, fill that section's summary and page links
- After leaf sections at one level are done, move up to their parent sections — fill summaries and sub-section links
- After all sections are complete, fill the book-level summary using the section summaries as context

**⚠️ Batch-fill all leaf sections first, then all parent sections.** Doing them one-by-one is impractical at scale (IMPLDDD had 231 sections). Use a Python script that:

1. Reads all SECTION files and builds a parent-child tree from their IDs (e.g., `SECTION2.1.1` → parent `SECTION2.1`)
2. Generates short Chinese summaries based on the section name and context
3. For leaf sections: replaces second `<待补充>` with a `- [[PAGE{N}]]` link list spanning the section's page range
4. For non-leaf sections: replaces second `<待补充>` with a `- [[SECTION{N.M}]]` sub-chapter link list
5. Writes all files in one pass

The script pattern is in `references/section-filling.md`.

After all sections are filled, update the book-level `# 摘要` to cover the full book structure, not just the first chapter.

## Technical Implementation Details

### PDF Reading Setup

Install dependencies into a dedicated venv (NOT hermes-agent's venv):

```bash
~/venv/mdbook/bin/pip install pymupdf marker-pdf
```

Always use this venv for MDBOOK work. `pymupdf` (imported as `fitz`) is the primary PDF reader; `marker-pdf` is the OCR fallback.

For mapping PDF absolute pages to printed book page numbers, see `references/pdf-page-mapping.md`.

### Text Extraction Strategy

**⚠️ REVISED — pymupdf is the PRIMARY extraction source; vision is for VERIFICATION only.**

Previous guidance said "use vision NOT pymupdf." This was wrong in practice. The `vision_analyze` tool has a hidden output token limit. On dense pages (400+ English words), the transcription SILENTLY truncates — bottom 20-40% of page content (callout boxes, bullet lists, closing paragraphs) is lost with no error indication. Every "vision-translated" page in IMPLDDD Ch1 was missing bottom content. PAGE1 had 1 of 7 roadmap bullets.

**pymupdf `get_text("text")` has NO token limit.** It captures every word on the page, including callout boxes, sidebars, and code. The only content it may miss is text rendered as vector graphics/illustrations — but those are rare compared to the consistent 20-40% loss from vision truncation.

**Mandatory workflow per page:**
1. Extract full text: `text = page.get_text("text")` — complete, no truncation
2. Render: `page.get_pixmap(dpi=200).save("/tmp/page.png")`
3. Verify bottom only: `vision_analyze(question="Read the last 5 lines verbatim. Any callout boxes, sidebars, or diagram text missing in the bottom half?")`
4. Translate from verified pymupdf text faithfully into Chinese
5. Spot-check: every 5-7 pages, do full vision comparison to catch systematic issues

For scanned PDFs where `get_text("text")` returns garbage, use two-pass vision:
- Pass 1: top 50% of page
- Pass 2: bottom 50% of page
- Merge both, then translate

**pymupdf usage:**
- **Primary text extraction** — complete, reliable, no token limits
- **Structural extraction** — TOC hierarchy via `get_toc()`, page ranges, bookinfo fields
- **Image extraction** — `get_images(full=True)` + clip-based for vector graphics
- **Page rendering** — for vision verification input

### Image Extraction

**Two approaches, in order of preference:**

**A. Direct extraction** (for embedded raster images):

```python
for img_index, img in enumerate(page.get_images(full=True)):
    xref = img[0]
    base_image = doc.extract_image(xref)
    if base_image["width"] < 50 or base_image["height"] < 50:
        continue  # skip tiny icons/decorative elements
    image_bytes = base_image["image"]
    ext = base_image["ext"]
    with open(f"FIG/{BOOK}-FIG-{section}-{img_index+1}.{ext}", "wb") as f:
        f.write(image_bytes)
```

**⚠️ Bulk extraction pattern — scan ALL body pages in one pass.** Rather than extracting images one page at a time, loop through all PDF pages, collect significant images (width > 100, height > 100), and batch-save with per-chapter sequencing:

```python
ch_counters = {}
for pdf_page in range(body_start, doc.page_count):
    printed = pdf_page - OFFSET
    ch = get_chapter(printed)  # map page number to chapter number
    if ch not in ch_counters:
        ch_counters[ch] = 1
    for img in page.get_images(full=True):
        w, h = base.get("width", 0), base.get("height", 0)
        if w > 100 and h > 100:
            seq = ch_counters[ch]
            fname = f"FIG/{BOOK}-FIG-{ch}-{seq}.{ext}"
            save image
            ch_counters[ch] += 1
```

Then batch-add `![[../FIG/...]]` references to the corresponding PAGE files. This is MUCH faster than extracting figure-by-figure with vision verification. In IMPLDDD, 80 images across 14 chapters were extracted in one script pass (~0.5s).

**⚠️ Not all extracted images are content figures.** `get_images()` also returns chapter-start decorations, inline icons, and repeated template elements (often ~300×119 pixels). These appear at chapter boundaries and should be skipped. Content figures typically have widths of 200–400px. **After bulk extraction, MUST visually spot-check at least 5-8 images across different chapters** — render the source page and the extracted image side-by-side with `vision_analyze` to confirm each is an actual content figure, not a decoration. In IMPLDDD, the 300×119 repeat images at chapter ends were template dividers, not content.

**B. Clip-based extraction** (for vector graphics, inline drawings, or figures not embedded as images):

1. Render the full PDF page and use `vision_analyze` to locate the figure:
   ```python
   pix = page.get_pixmap(dpi=200)
   pix.save("/tmp/page_N.png")
   # Then: vision_analyze(image_url="/tmp/page_N.png", 
   #   question="Where exactly is the figure on this page? Give proportions (top%/bottom%/left%/right%)")
   ```

2. Convert proportions to `fitz.Rect` and clip:
   ```python
   w, h = page.rect.width, page.rect.height
   # Example: figure at top 24%, bottom 68%, left 12%, right 88%
   clip = fitz.Rect(w*0.12, h*0.24, w*0.88, h*0.68)
   pix = page.get_pixmap(clip=clip, dpi=200)
   pix.save(f"FIG/{BOOK}-FIG-{section}-{seq}.png")
   ```

3. **Verify the clipped image** — render the original page again and use `vision_analyze` to compare:
   - "Compare this clipped image with the original page. Is the figure complete? Any adjacent text bleeding in? Anything cropped off?"

**⚠️ CRITICAL: Image Verification**

Extracted images MUST be verified against the original book page. Never trust automated extraction alone — it can miss images, extract wrong regions, or include extra content.

**Verification workflow:**

1. Render the original PDF page at high DPI:
```python
pix = page.get_pixmap(dpi=200)
pix.save("/tmp/original_page.png")
```

2. Use `vision_analyze` to compare extracted image with the original page:
   - Question: "Compare this extracted image with the original page. Is this the complete figure from the book? Is anything missing, cropped, or extra?"
   - Pass both the original page screenshot and the extracted image

3. If the image is wrong:
   - **Missing parts:** Try a larger crop region
   - **Extra content:** Tighten the crop region
   - **Wrong image entirely:** Check `img_index` — pymupdf lists images in PDF internal order, which may not match visual page order
   - **Image not found in extraction:** The image may be vector graphics or inline drawings — render the page region directly:
```python
clip = fitz.Rect(x0, y0, x1, y1)
pix = page.get_pixmap(clip=clip, dpi=200)
pix.save("FIG/cropped_figure.png")
```

4. **Image boundaries must be exact** — no extra margin showing adjacent text, no cropping into the figure itself. If the image spans multiple columns or has a caption, decide whether the caption stays with the image or becomes page text.

**Pitfall:** Some PDF images are embedded as multiple small tiles stitched together by the PDF renderer. pymupdf extracts each tile separately. In this case, rendering the region directly (clip + get_pixmap) is more reliable than extracting embedded images.

### Section-Level Content

For leaf sections, the `# 页面` field lists every page in that section:

```
# 页面

- [[MDBOOK260503-IMPLDDD-PAGEXVII]]
- [[MDBOOK260503-IMPLDDD-PAGEXVIII]]
```

For non-leaf sections, `# 子章节` lists sub-sections while `# 页面` holds placeholder for eventual page links:

```
# 子章节

- [[MDBOOK260503-IMPLDDD-SECTION1.1]]
- [[MDBOOK260503-IMPLDDD-SECTION1.2]]

# 页面

<待补充>
```

**Pitfall:** Don't skip pages. If section 1.1 covers pages 5-12, all 8 pages must appear in the page list. Missing pages break the prev/next chain.

## Common Pitfalls

### Process & Setup

1. **Skipping sync before/after editing.** Always sync the Obsidian vault before and after any MDBOOK file writes. Unsynced edits cause merge conflicts.

2. **Writing page summaries.** Pages do NOT need summaries. Summaries live at the section and book level only.

3. **Wrong venv.** Install pymupdf/marker-pdf to a dedicated venv (e.g., `~/venv/`). Do NOT install into hermes-agent's venv.

4. **Skill drifts from design doc.** The master design is `[[E2605272 PDF转Markdown思路]]`. When the user updates E2605272, this skill MUST be re-aligned immediately. After every E2605272 edit, compare templates line-by-line.

### Structure & Navigation

5. **Broken prev/next chains.** Every page's `上一页`/`下一页` must be bidirectional. Same for SECTION `上一章`/`下一章`. Verify after every structural change.

6. **Using PDF absolute page numbers instead of printed page numbers.** Map PDF pages to printed pages (Roman for frontmatter, Arabic for body) before writing any page file name or `页码范围`.

7. **Creating only top-level chapter skeletons.** Step 2 must create the FULL section tree — every L2/L3 sub-section down to leaves. Parse the TOC tree completely.

8. **Book TOC entries must ALL include `-[[PAGE起始页]]`.** Every `# 目录` entry: `- ***X.Y-名称*** - [[SECTION]]-[[PAGE起始页]]`. The PAGE link is NOT optional.

9. **SECTION navigation fields corrupted with prose.** `上一章`/`下一章` must be `无` or `[[SECTION]]` wiki-links — never Chinese summary text. Run a programmatic scan to catch prose in nav fields.

10. **Missing or incomplete SECTION files.** Two variants: (A) SECTION file never created → dead links in TOC. (B) SECTION exists but `# 页面` has placeholder text (`本节围绕...`) instead of `- [[PAGE{N}]]` links. After Step 2, verify every `[[SECTION]]` reference has a matching file AND wiki-link page list.

11. **Section deletion cascades.** Removing a chapter has three ripple effects: (a) remove TOC entry, (b) reconnect prev/next across the gap, (c) scan surviving PAGE files for in-text `[[PAGE...]]` references to deleted pages.

12. **MDBOOK directory organization — use `BOOKS/` subdirectory.** Nest each book under `Resource/MDBOOK/BOOKS/` for multi-book vaults. Create a lightweight `R<YYMMDDNN>` adapter file linking to the book root.

### Text Extraction & Content

13. **⚠️ vision_analyze silently truncates — pymupdf is PRIMARY; vision is VERIFICATION only.** The `vision_analyze` tool has a hidden output token cap. On dense pages, bottom 20-40% of content is silently lost. **pymupdf `get_text("text")` has NO token limit.** Mandatory workflow: (1) extract full text with pymupdf, (2) render and verify bottom 5 lines with vision, (3) translate from pymupdf text, (4) spot-check every 5-7 pages. For scanned PDFs, use two-pass vision (top half + bottom half separately).

    **Frontmatter pages are the HIGHEST-RISK zone.** Preface, Acknowledgments, and Guide to This Book sections are dense narrative prose (400-500 English words/page) — exactly the profile that triggers vision truncation. In IMPLDDD, EVERY frontmatter page translated via vision-only was damaged: PAGEXX lost 80%, PAGEXXI lost 60%, PAGEXXII lost 50%, PAGEXXVIII lost 40%, PAGEXXXI lost 75%. After any bulk translation, audit ALL frontmatter pages first — they're the most likely to have silent bottom-content loss. See pitfall #38 for the word-count ratio detection method.

14. **Putting page content into SECTION files instead of PAGE files.** SECTION files are NAVIGATION NODES only — they link to child nodes, never contain actual page text. Every physical book page gets its own `PAGE{N}.md` with `# 内容`. SECTION `# 页面` lists `- [[PAGE{N}]]` wiki-links, not inline text.

15. **Content language mismatch.** The user's vault is Chinese. All `# 内容` in PAGE files must be translated to Chinese. Summaries in SECTION files are also Chinese.

16. **⚠️ Page content MUST be cleaned of all printer's marks before translation.** Strip three non-content elements from EVERY page: (a) bare page numbers, (b) running headers (chapter names at top/bottom), (c) book title/author footers. Then translate faithfully — do NOT write content from your own domain knowledge.

17. **Cover pages return empty text from pymupdf.** Start bookinfo extraction from page 1 (second page) or skip pages returning empty.

18. **⚠️ Chapter/section titles MUST use `##` headings with original typography.** `第2章 领域、子域` → `## 第2章 领域、子域与限界上下文`. Preserve original book emphasis with `***` (bold+italic): `## ***用领域驱动设计着陆***`. Mid-flow pages inherit the previous page's heading or use none. **Chapter-number references go OUTSIDE bold:** `**统一语言**（1）` not `**统一语言（1）**` — the `（N）` is a cross-reference, not part of the term name.

### Translation

19. **⚠️ NEVER condense or abbreviate translations.** ~400 English words ≠ ~250 Chinese chars. Every paragraph, bullet point, and code snippet must be present. Verify paragraph counts match between source and output.

20. **English content in Chinese PAGE files — not all English is an error.** Expected English: (a) code blocks, (b) proper names (Vaughn Vernon, SaaSOvation), (c) community-standard terms (DDD, CQRS, REST). Only prose, titles, captions, and callout boxes should be fully Chinese.

21. **Backmatter pages need distinct handling.** INDEX pages preserve original English terms alongside Chinese (e.g., "Aggregate, 347-388"). Promotional pages get a brief description rather than full translation.

22. **⚠️ Page layout must match the original book.** Do NOT reorder content within a page. If the original book places the table at the top, put it at the top — not after prose text. If an image is embedded mid-paragraph, keep it there. The only permitted transformations: stripping printer's marks (#16), merging broken lines, converting tables/images to Markdown, and translating prose. Reordering paragraphs, tables, or figures relative to each other changes the author's intended reading flow.

### Cross-Page Continuity

22. **⚠️ NEVER use translator artifacts: "(接上页)", "(续)", "待续".** These artificial markers make pages look like a rough draft. The original book has none of these. (a) **"(接上页)"**: Instead, carry the last phrase from the previous page's ending to the current page's opening. Example: PAGEXIX ends `尽管我看不清她们的眼睛。` PAGEXX starts `(接上页)...交流。` → fix: `尽管我看不清她们的眼睛，也无法和她们交流。如果我朝着飞机窗外大喊...`. After fixing, scan for zero residual. (b) **"（续）" in headings**: Only the FIRST page of a chapter/section gets a `##` heading. Continuation pages drop `##` entirely — just use `###` sub-headings or plain content. Strip `（续）` from all headings. (c) **"*待续*"**: Remove. The only exception is table titles mirroring the original book's "(Continued)" convention. In IMPLDDD, ~50 heading-level （续） and 7 (接上页) were removed.

23. **Cross-page content continuity is as critical as individual page completeness.** After any repair or bulk translation, verify PAGE N's last paragraph flows into PAGE N+1's first. A content GAP between pages is worse than an overlap. **Truncation clusters — when one page in a section is truncated, check ALL neighboring pages in the same chapter.** In IMPLDDD, fixing PAGEXX revealed PAGEXXI was also missing 60% of its content.

### Section & Navigation Integrity

24. **Non-leaf SECTION `# 页面` must list ALL underlying pages.** SECTION2 (pages 43-86) must have 44 `- [[PAGE{N}]]` links — not `<待补充>`, not sub-section links. `# 子章节` provides hierarchy; `# 页面` provides direct page access.

25. **SECTION-boundary navigation breaks easily.** Frontmatter→Ch1 transition is a weak point. Verify last frontmatter page's `next` → PAGE1, and PAGE1's `prev` → last frontmatter page.

26. **String-substring matching trap in navigation audits.** When scanning for broken links, use boundary-aware matching. `[[PAGEX]]` (exact) ≠ `PAGEXVII` (false positive). Use `\[\[MDBOOK...-PAGEX\]\]` not substring `PAGEX`.

### Bulk Operations

27. **Create ALL PAGE files in the target range before translating.** Immediately create every PAGE file with `# 链接` and `<待补充>`, then extract and translate. Prevents "10 pages were never created" bug.

28. **Bulk translation delegation pattern.** For 20+ pages, use `delegate_task` with parallel subagents (~40-50 pages each; 60+ pages hits timeout). Provide pymupdf source text, file paths, and PAGE template. After completion: (a) spot-check boundary pages, (b) fix overlaps, (c) verify prev/next links, (d) sync. See `references/parallel-translation-pattern.md`.

29. **Post-timeout partial work recovery — scan before retrying.** When a subagent times out, completed pages ARE valid on disk. Scan for remaining `<待补充>`, delegate only the still-untranslated sub-range.

30. **⚠️ Verify page-offset mapping per chapter.** After bulk translation, run a Python script to extract printed page numbers from pymupdf and confirm file-name match. Mismatch = content written to wrong PAGE files.

31. **execute_code is unreliable for writing translated content.** Use `write_file` for translation output — it handles arbitrary content reliably. Use `execute_code` only for PDF rendering, pymupdf extraction, offset verification, and auditing.

32. **⚠️ User-directed hybrid fast-then-verify workflow.** When user requests efficiency ("先用快速方式转换，转换后视觉验证"): (1) batch-render all pages, (2) extract all text via pymupdf, (3) translate in bulk with `write_file`, (4) visual spot-check 3-5 pages, (5) fix mismatches. This is a user-acknowledged tradeoff, not a replacement for the pymupdf-primary workflow (#13).

### Images & Figures

33. **Skipping image extraction or verification.** Every page must be checked for images. After extracting, always verify against original with `vision_analyze`. Invisible errors (stray text, cropped edges, wrong figure) silently degrade quality.

34. **⚠️ `get_images()` produces decorative tiles, NOT book figures.** `get_images()` extracts embedded raster metadata — tiny decorative elements (8-17KB, ~300px), not actual diagrams. Real book figures are VECTOR GRAPHICS. Detection: any extracted image under 20KB or under 400×400px is decorative. Workflow per chapter: (1) render all pages, (2) scan with vision to locate figures, (3) clip-extract with `get_pixmap(clip=...)`, (4) visually verify, (5) embed in correct PAGE. Expect 2 rounds per figure. See `references/parallel-vision-figure-scan.md`.

35. **Image clip extraction uses proportional coordinates, not pixel guesses.** Use vision to find bounding box as fractions, compute `fitz.Rect`: `fitz.Rect(w*0.17, h*0.24, w*0.78, h*0.52)`. Expect 2-3 refinement rounds per image. Common failures: text bleeding, over-crop, blank extraction, wrong content.

35a. **⚠️ Figure cropping MUST produce equal margins on all four sides.** After clip-extraction, the figure should have uniform tiny white margins (top/bottom/left/right). Two-phase approach: (1) render the figure region with tight clip, (2) convert to PIL, detect ink bounds via grayscale threshold, crop to content, add equal padding via `ImageOps.expand(border=N, fill='white')`. Target ~50px at 4× zoom (~12pt). After cropping, visually verify — if the figure has top-heavy elements (labels, arrows), add 15-20px extra top padding to visually balance the inherently asymmetric content. See `references/equal-margin-figure-crop.md`.\n\n36. **FIG↔PAGE cross-reference integrity.** Every FIG file must be referenced in exactly one PAGE file's `![[../FIG/...]]`. Every `![[` reference must point to an existing FIG file. Run audit after image extraction.

37. **FIG files must be embedded in the correct PAGE that contains the figure.** Bulk extraction puts files in `FIG/` but `![[../FIG/...]]` must appear in the PAGE whose content describes that figure. Cross-reference every "图X.Y" mention in translated text against FIG files.

### Auditing & Verification

38. **⚠️ After bulk translation, run a systematic content audit on ALL pages.** Two-part audit: (A) scan every PAGE for truncation signatures (abrupt endings, mid-character cuts, short last lines). (B) scan all pages for prev/next links pointing to deleted/non-existent pages. Fix workflow: locate source PDF → extract with pymupdf → translate faithfully → `write_file` → verify cross-page continuity. See `references/content-audit.md`.

    **Supplement — word-count ratio detection:** Programmatic truncation scanning is aggressive but a simpler mass-triage is comparing PDF English word count against MDBOOK Chinese character count. Healthy ratio: Chinese chars ≈ 0.5–0.7 × English words. A ratio above 0.85 (e.g., 489 English words → only 390 Chinese chars) reliably indicates 30–75% of the page was silently dropped. Run this scan across ALL pages after any bulk translation — it catches vision-truncated pages that pass syntax-level truncation scans (the page "looks complete" but half the paragraphs are missing). In IMPLDDD, this caught PAGEXXVIII (ratio 1.25) and PAGEXXXI (ratio 1.30) which both appeared syntactically complete.

## Verification Checklist

- [ ] All `[[wikilink]]` targets exist as files
- [ ] Page sequence is continuous (no gaps in page numbers)
- [ ] prev/next links are bidirectional (A→B and B←A)
- [ ] Book file's `# 目录` covers all top-level sections
- [ ] Every TOC entry has both `-[[SECTION]]` and `-[[PAGE起始页]]`
- [ ] Every section has a summary
- [ ] No page has a summary
- [ ] All extracted images verified against original PDF pages with `vision_analyze`
- [ ] Every body PAGE file was translated from pymupdf text, with vision verification of bottom content
- [ ] Paragraph counts match between pymupdf text and Chinese translation per page
- [ ] Vault synced after all file writes
- [ ] Visual spot-check: at least 3-4 pages from different chapter sections verified for completeness
- [ ] Page boundaries: no sentence overlap between consecutive pages (re-read adjacent PAGE files to check)
- [ ] Subagent output: after parallel delegation, verified no content was placed on wrong PAGE files
- [ ] **Systematic content audit:** programmatic scan of ALL pages for truncation signs (see `references/content-audit.md`)
- [ ] **Word-count ratio scan:** Chinese chars ÷ English words < 0.5 flags possible truncation; < 0.3 flags severe loss
- [ ] **Navigation audit after structural changes:** after deleting pages or reordering, scan all remaining pages for broken prev/next links
- [ ] **Cross-page continuity:** PAGE N's ending flows into PAGE N+1's beginning — no content gaps between consecutive pages
- [ ] **No translator artifacts:** zero `(接上页)`, `（续）` in headings, `*待续*` across all files; all cross-page breaks use natural phrase carry-forward
- [ ] **FIG↔PAGE cross-reference:** every FIG file is referenced in a PAGE file, every `![[../FIG/` reference points to an existing file
- [ ] **Section boundaries:** frontmatter→Ch1 and ChN→ChN+1 prev/next are bidirectional and correct
- [ ] **SECTION navigation fields:** every `上一章`/`下一章` in every SECTION file is either `无` or a `[[SECTION]]` wiki-link (not prose/summary text)
- [ ] **Non-leaf SECTION `# 页面` completeness:** every parent SECTION's `# 页面` lists ALL underlying PAGE files in its `页码范围`, not `<待补充>` and not sub-section links. SECTION2 (pages 43-86) must have 44 PAGE links.
- [ ] **Content-internal links:** no in-text `[[PAGE...]]` references point to deleted pages
- [ ] **Printer's marks stripped:** no bare page numbers, running headers, or book-title footers in any PAGE file's `# 内容`
- [ ] **Heading correctness:** chapter/section titles use `##` headings; no wrong section heading on mid-flow pages
- [ ] **FIG placement:** every figure mentioned in translated text has `![[../FIG/...]]` embed at correct page position
- [ ] **FIG quality verified:** every extraction visually checked against original; no bottom-cut captions, no clipped annotations; uniform tiny margins on all 4 sides with content centered

## Related References

- `references/page-file-concrete-example.md` — Concrete BEFORE vs AFTER examples for PAGE files, translation completeness, TOC format, and image clipping
- `references/pdf-page-mapping.md` — How to map PDF absolute page numbers to book printed page numbers
- `references/batch-rendering.md` — Python pattern for rendering 10-40 pages in one pass
- `references/visual-translation-workflow.md` — Hybrid pymupdf + vision verification workflow, two-pass vision patterns
- `references/image-extraction-failures.md` — Catalog of image extraction failure modes (text bleeding, over-crop, blank extraction, wrong content)
- `references/parallel-translation-pattern.md` — Parallel subagent delegation for bulk translation: batch sizing, timeout recovery, source extraction, skeleton creation, and post-completion verification
- `references/section-filling.md` — Batch-fill SECTION summaries and page/sub-chapter links for 200+ sections using Python scripts with parent-child tree walking
- `references/parallel-vision-figure-scan.md` — Parallel subagent pipeline for figure scanning: batch render → 3-4 subagent vision scan → aggregate → batch clip extract. Proven on IMPLDDD (600+ pages, ~30 min for 14 chapters)
- `references/equal-margin-figure-crop.md` — Two-phase Python pattern for equal-margin figure cropping: tight PDF clip → PIL ink detection → uniform `ImageOps.expand` → visual compensation for asymmetric content
- `references/search-files-roman-numeral.md` — Workaround for `search_files` failing to match Roman numeral filenames (PAGEXVII, etc.); use `ls` in terminal instead

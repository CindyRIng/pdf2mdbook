---
name: mdbook-conversion
description: Use when converting PDF books to Obsidian MDBOOK format. Covers directory structure, file templates, 4-step bottom-up workflow, text extraction strategies (pymupdf/OCR/vision), image extraction with visual verification against original, and common pitfalls.
version: 2.8.0
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
5. **Stray `\t` characters** — remove trailing tab characters at end of lines (e.g., `\t\n`). These are formatting artifacts from PDF extraction.

**Content formatting rules — MUST apply after cleaning:**

| Original book feature | MDBOOK Markdown mapping |
|---|---|
| Callout boxes (sans-serif, indented, bounded by horizontal rules) | `> ` blockquote |
| Emphasized statements / pull quotes | `> ` blockquote |
| Whiteboard exercises / discussion questions | `> ` blockquote |
| Standard instructional prose (serif body text) | Plain text (no `> `) |
| Sub-section headings within pages | `### ***标题***` (bold-italic) |
| Chapter titles | `## 第X章 章节名` |
| Figure captions | `***图X.Y*** *描述文字*` (bold-italic number + italic description) |
| Blockquote nested sub-content | `> \t内容` (tab-indented within blockquote) |
| Bullet lists within callout boxes | `> - item` (list inside blockquote) |
| Inline cross-chapter references | ` ***名称（N）*** ` (three-asterisk bold-italic wrapping name+number together, space before and after) |

**Determining blockquote vs plain text:** This requires comparing the translated page against the original PDF. Callout boxes are visually distinct in the original: they use a different typeface (sans-serif vs serif) and are bounded by horizontal rules above and below. When in doubt, render the original PDF page and use `vision_analyze` to determine which paragraphs are in callout boxes vs standard body text.

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
┌──────────────────────────────────────────────────────────────────────┐
│                         PDF → MDBOOK                                  │
├──────────┬──────────┬──────────┬──────────────┬────────────────────┤
│ 1.初始化  │ 2.建目录  │ 3.拆页面  │ 3.5.视觉验证  │ 4.补详情            │
│ 书籍骨架  │ 章节骨架  │ 内容提取  │ 每页强制比对  │ 自底向上            │
└──────────┴──────────┴──────────┴──────────────┴────────────────────┘
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
- Content pipeline for EVERY page — **in this exact order, no exceptions:**

  1. **Extract** English text from PDF with pymupdf `get_text("text")`
  2. **Verify extraction completeness BEFORE translating** — compare tail against PDF (see checkpoint below)
  3. **Translate** faithfully to Chinese
  4. **Write** `# 内容` in PAGE file

  All four steps are mandatory. Skipping translation or inventing content from domain knowledge both produce wrong results. **The verify-before-translate gate is critical:** do NOT pass extracted text to a subagent or translator until you have confirmed it contains the ENTIRE page. Subagents have no PDF access — they cannot detect truncation. If you hand them a partial extract, the missing content is invisible to them and becomes invisible to future audits.

- Start from the first page referenced in the TOC. For every page in the book's page range, create a PAGE file in `CONTENT/`.
- **Images:** Extract from PDF, crop via 8-step pipeline (see Image Extraction below), embed with `![[../FIG/...]]`.
- **Tables:** Convert to Markdown tables
- **Formulas:** Express in LaTeX (`$$...$$` or `$...$`)

**⚠️ MANDATORY GATE — Verify extraction completeness BEFORE translating (for EVERY page):**

Before translating ANY page, confirm the pymupdf text extraction captured the entire page:

1. Extract: `text = page.get_text("text")`
2. Render: `page.get_pixmap(dpi=200).save("/tmp/page_N.png")`
3. Compare the tail: read the last ~50 chars of the extracted text, then check the PDF render's bottom portion with `vision_analyze(question="Read the last 5 lines verbatim. Any content missing from the bottom?")`.
4. Only if the tail matches → proceed to translate. If not → the extraction is truncated; re-run or investigate.

**Why this gate must run BEFORE translation, not after:** Subagents and batch translators have no PDF access. If you hand them a truncated extract, the missing content is forever invisible. Running the tail check after writing (in Step 3.5) catches the error but requires re-extraction + re-translation of the entire page — wasted work. Running it as a pre-translate gate prevents the waste entirely.

**⚠️ After translating and writing, verify against original PDF (Step 3.5) before declaring any page "done."**

After all pages for a section are created, update the leaf SECTION's `# 页面` to list every PAGE file link.

### Step 3.5: Per-Page Visual Layout Verification (MANDATORY)

**⚠️ This step is NOT optional. PDF text extraction order ≠ visual page layout order. Skipping this step produces invisible errors.**

**Why mandatory:** PDF files store content in internal byte-stream order, not visual reading order. A figure at the TOP of a printed page may appear AFTER body text in pymupdf output. This affects ALL PDFs regardless of language or publisher.

**Two verification tiers:**

**Tier A — Pages with figures, callout boxes, or sidebars (full visual audit):**

1. Render the original book page at 150 DPI
2. Use `vision_analyze` to determine visual layout: "What appears at the top/middle/bottom? Exact order of ALL elements."
3. Rebuild MDBOOK PAGE matching visual order
4. Verify: figure count matches, figure position matches, blockquote boundaries match callout boundaries
5. Verify figure existence: every `![[../FIG/` reference must correspond to a figure that ACTUALLY APPEARS on the original book page. Render and visually confirm.
6. For callout boxes spanning page boundaries: `>` must carry across both pages.

**Tier B — Text-only pages (confirm Step 3 gate passed):**

For pages with no figures, callout boxes, or sidebars — the primary defense is the **Step 3 pre-translate gate** (tail alignment against PDF BEFORE translation). Tier B is a lightweight secondary confirmation:

1. Re-run the tail check: verify last ~50 chars of the written MDBOOK page align with the PDF page's bottom content.
2. Check cross-page flow: PAGE N's last line should naturally lead into PAGE N+1's first line.

If Tier B finds truncation, the root cause is a skipped Step 3 gate — fix the process, not just the page.

### Step 4: Bottom-Up Detail Population

- Start from the **leaf** (deepest) sections
- After reading all pages under a leaf section, fill that section's summary and page links
- After leaf sections at one level are done, move up to their parent sections
- After all sections are complete, fill the book-level summary

**⚠️ Batch-fill using a Python script** for 200+ sections. Pattern: read all SECTION files, build parent-child tree, generate summaries, fill page/sub-chapter links in one pass. See `references/section-filling.md`.

## Technical Implementation Details

### PDF Reading Setup

```bash
~/venv/mdbook/bin/pip install pymupdf marker-pdf
```

Always use a dedicated venv. `pymupdf` is primary; `marker-pdf` is OCR fallback.

### Text Extraction Strategy

**⚠️ pymupdf is PRIMARY; vision is VERIFICATION only.** Vision has a hidden token cap that silently truncates bottom 20-40% of dense pages. pymupdf has NO token limit.

**Mandatory workflow per page:**
1. Extract full text with pymupdf
2. Render and verify bottom 5 lines with vision
3. Translate from verified pymupdf text
4. Spot-check every 5-7 pages

For scanned PDFs, use two-pass vision (top half + bottom half separately).

### Image Extraction

**Two approaches:**

**A. Direct extraction** — for embedded raster images (rare in technical books).

**B. Clip-based extraction** — for vector graphics (the common case). Full 8-step pipeline:

1. Locate figure pages via text search for "Figure X.Y"
2. Render candidate pages at 300 DPI, use vision to find position as proportions
3. Refine boundaries with text block analysis
4. Set crop boundaries: `fitz.Rect(w*left, h*top, w*right, h*bot)`
5. PIL ink detection + equal margins (50px uniform padding)
6. Exclude horizontal separator rules
7. Visual compensation for asymmetric content
8. Vision verify every extracted figure (2-4 rounds per image)

**⚠️ After ANY batch extraction, run dimension check:** if `PIL.Image.open(f).size[1] > 800`, the file is a full-page render, not a cropped figure. Route through the 8-step pipeline. All 11 Ch4 figures were caught by this check.

Full pattern in `references/equal-margin-figure-crop.md`.

## Common Pitfalls

### Pitfall Index

| # | Category | Topic |
|---|----------|-------|
| 0 | Process | Always load THIS skill |
| 1 | Process | Skipping sync |
| 2 | Process | Writing page summaries |
| 3 | Process | Wrong venv |
| 4 | Process | Skill drifts from design doc |
| 5 | Process | Chapter rebuild: page-by-page |
| 6 | Structure | Broken prev/next chains |
| 7 | Structure | PDF absolute vs printed page numbers |
| 8 | Structure | Only top-level chapter skeletons |
| 9 | Structure | Book TOC missing PAGE links |
| 10 | Structure | SECTION nav fields corrupted |
| 11 | Structure | Missing SECTION files |
| 12 | Structure | Section deletion cascades |
| 13 | Structure | MDBOOK dir: use BOOKS/ |
| 14 | Layout | Extraction order ≠ visual order |
| 15 | Layout | Layout reversal is systemic |
| 16 | Layout | Blockquote boundary drift |
| 17 | Layout | Figures on wrong page |
| 18 | Layout | Step 3.5 non-compliance is root cause |
| 19 | Layout | Content truncation on text-only pages |
| 20 | Text | vision_analyze silently truncates |
| 21 | Text | Content in SECTION files |
| 22 | Text | Language mismatch |
| 23 | Text | Printer's marks not stripped |
| 24 | Text | Cover pages return empty |
| 25 | Text | Chapter/section title formatting |
| 26 | Text | Composite figure code duplication |
| 27 | Translation | NEVER condense abbreviations |
| 28 | Translation | English in Chinese pages |
| 29 | Translation | Backmatter handling |
| 30 | Translation | Page layout must match original |
| 31 | Cross-Page | Translator artifacts (接上页/续/待续) |
| 32 | Cross-Page | Cross-page continuity |
| 33 | Cross-Page | Paragraphs after heading |
| 34 | Navigation | Non-leaf SECTION page list |
| 35 | Navigation | SECTION-boundary navigation |
| 36 | Navigation | String-substring matching trap |
| 37 | Bulk | Create ALL PAGE files first |
| 38 | Bulk | Delegation + pre-translate gate |
| 39 | Bulk | Post-timeout recovery |
| 40 | Bulk | Page-offset mapping |
| 41 | Bulk | execute_code unreliable for translation |
| 42 | Bulk | Hybrid fast-then-verify |
| 43 | Images | Skipping extraction/verification |
| 44 | Images | get_images() = decorative tiles |
| 45 | Images | Image clip: proportional coordinates |
| 46 | Images | Figure cropping: equal margins |
| 47 | Images | FIG↔PAGE cross-reference integrity |
| 48 | Images | FIG files on correct PAGE |
| 49 | Images | Full-page renders saved as FIG |
| 50 | Audit | Systematic content audit after bulk |

### Process & Setup

0. **⚠️ Always load THIS skill for ANY MDBOOK task — inspection, repair, or conversion.** The `obsidian-vault` skill's `references/mdbook-conventions.md` covers format conventions only — not page inspection, figure extraction, or layout verification.

1. **Skipping sync before/after editing.** Always sync the Obsidian vault before and after any MDBOOK file writes.

2. **Writing page summaries.** Pages do NOT need summaries. Summaries live at the section and book level only.

3. **Wrong venv.** Install pymupdf/marker-pdf to a dedicated venv. Do NOT install into hermes-agent's venv.

4. **Skill drifts from design doc.** The master design is `[[E2605272 PDF转Markdown思路]]`. Re-align immediately after E2605272 edits.

5. **⚠️ Chapter rebuild/repair: page-by-page, not multi-step plans.** When the user says "重整": start with PAGE{N}, process completely (extract → verify → translate → visual verify → write), then PAGE{N+1}. No skipping, no delegating batches.

### Structure & Navigation

6. **Broken prev/next chains.** Every page's `上一页`/`下一页` must be bidirectional. Verify after structural changes.

7. **Using PDF absolute page numbers instead of printed page numbers.** Map PDF pages to printed pages before writing file names or `页码范围`.

8. **Creating only top-level chapter skeletons.** Step 2 must create the FULL section tree — every L2/L3 sub-section down to leaves.

9. **Book TOC entries must ALL include `-[[PAGE起始页]]`.** Every entry: `- ***X.Y-名称*** - [[SECTION]]-[[PAGE起始页]]`. The PAGE link is NOT optional.

10. **SECTION navigation fields corrupted with prose.** `上一章`/`下一章` must be `无` or `[[SECTION]]` wiki-links — never Chinese summary text.

11. **Missing or incomplete SECTION files.** Verify every `[[SECTION]]` reference has a matching file with wiki-link page list.

12. **Section deletion cascades.** Three ripple effects: remove TOC entry, reconnect prev/next, scan for `[[PAGE...]]` references to deleted pages.

13. **MDBOOK directory organization — use `BOOKS/` subdirectory.** Nest each book under `Resource/MDBOOK/BOOKS/`.

### Page Layout Verification

14. **⚠️ PDF text extraction order ≠ visual layout order.** Render every page and verify layout visually. A figure at the TOP may appear AFTER text in pymupdf output. Reconstruct MDBOOK matching visual order. See `references/ch3-before-after-examples.md` case #1 for PAGE99 FIG3-5 example.

15. **⚠️ Layout reversal is SYSTEMIC — finding one triggers a full-chapter scan.** The PDF internal stream order that causes reversal is a book-wide property. When one page has figure/text reversal, scan ALL remaining pages in that chapter.

16. **⚠️ Blockquote boundary drift — callout-box text markers leak into plain body text.** Verify every `>` blockquote by rendering the original PDF and confirming that paragraph is visually inside a callout box. Example: IMPLDDD PAGE97 had plain body paragraphs wrongly marked as blockquote.

17. **⚠️ Figures appearing on the wrong page.** After embedding figures, render every page and confirm: the set of figures on the original page exactly matches the set of `![[../FIG/` references. See `references/ch3-before-after-examples.md` case #2 for PAGE99/101/111 examples.

18. **⚠️ Step 3.5 non-compliance is the ROOT CAUSE of nearly all layout errors.** Pitfalls 14–17 all share one root cause: Step 3.5 was skipped. Finding one error is a signal to audit ALL surrounding pages. See `references/ch3-before-after-examples.md` for the error frequency summary — 5 layout reversals, 3 extra figures, 2 blockquote drifts, all from skipping Step 3.5.

19. **⚠️ Content truncation on text-only pages.** Pure dialogue or prose pages without figures can still have their bottom content silently dropped. Detection: page ends mid-sentence, next page opens with a response assuming unstated prior context. Real examples: PAGE116 (40% lost, Mitchell's Ports/Adapters explanation), PAGE117 (40% lost, CQRS justification + distributed processing dialogue), PAGE118 (20% lost, Event Sourcing + interview conclusion).

### Text Extraction & Content

20. **⚠️ vision_analyze silently truncates — pymupdf is PRIMARY.** Vision has a hidden token cap; pymupdf `get_text("text")` has NO token limit. Frontmatter pages are highest-risk: in IMPLDDD, PAGEXX lost 80%, PAGEXXI lost 60%.

21. **Putting page content into SECTION files instead of PAGE files.** SECTION files are navigation nodes only. Every physical book page gets its own `PAGE{N}.md`.

22. **Content language mismatch.** All `# 内容` must be translated to Chinese.

23. **⚠️ Page content MUST be cleaned of all printer's marks before translation.** Strip bare page numbers, running headers, and book title/author footers from every page.

24. **Cover pages return empty text from pymupdf.** Start bookinfo extraction from page 1 or skip empty pages.

25. **⚠️ Chapter/section titles MUST use `##` headings.** Preserve original book emphasis. Cross-chapter references: ` ***名称（N）*** ` — three asterisks wrapping name+number together, space before and after.

26. **⚠️ Composite figures: do NOT extract embedded code/text as separate content.** When a figure is a composite diagram with code listings, the `![[FIG]]` already contains all of that. Do NOT duplicate as a code block.

### Translation

27. **⚠️ NEVER condense or abbreviate translations.** Every paragraph, bullet, and code snippet must be present.

28. **English content in Chinese PAGE files — not all English is an error.** Expected English: code blocks, proper names, community-standard terms (DDD, CQRS, REST). Only prose and captions must be Chinese.

29. **Backmatter pages need distinct handling.** INDEX pages preserve original English terms; promotional pages get brief descriptions.

30. **⚠️ Page layout must match the original book.** Do NOT reorder content within a page. Only permitted transformations: stripping printer's marks, merging broken lines, converting tables/images, and translating prose.

### Cross-Page Continuity

31. **⚠️ NEVER use translator artifacts: "(接上页)", "(续)", "待续".** The original book has none. Instead, carry the last phrase from PAGE N's ending to PAGE N+1's opening for natural flow.

32. **Cross-page content continuity is as critical as individual page completeness.** After any repair, verify PAGE N flows into PAGE N+1. Truncation clusters — when one page is truncated, check all neighboring pages.

33. **⚠️ Paragraphs following a heading stay on the SAME PAGE as the heading.** When a heading is near bottom of a page and followed by a body paragraph, the paragraph belongs on the heading's page — not at the top of the next page. Example: PAGE107's heading was orphaned when its paragraph moved to PAGE108.

### Section & Navigation Integrity

34. **Non-leaf SECTION `# 页面` must list ALL underlying pages.** SECTION2 (pages 43-86) must have 44 `- [[PAGE{N}]]` links.

35. **SECTION-boundary navigation breaks easily.** Verify frontmatter→Ch1 and ChN→ChN+1 prev/next are bidirectional.

36. **String-substring matching trap in navigation audits.** Use boundary-aware matching: `\[\[MDBOOK...-PAGEX\]\]` not substring `PAGEX`.

### Bulk Operations

37. **Create ALL PAGE files in the target range before translating.** Prevents "10 pages were never created" bug.

38. **Bulk translation delegation pattern.** For 10+ pages, use `delegate_task` with parallel subagents (~8-12 pages each). **⚠️ The Step 3 pre-translate gate APPLIES to subagents too.** Verify extraction completeness BEFORE handing to subagents — they have no PDF access and cannot detect truncation. After completion: spot-check boundaries, verify prev/next, run Tier B tail-alignment on ALL text-only pages, sync. Critical learning from Ch4 rebuild: 9 pages were silently truncated because extracts were handed to subagents unverified.

39. **Post-timeout partial work recovery — scan before retrying.** Completed pages ARE valid on disk. Scan for remaining `<待补充>`, delegate only the still-untranslated sub-range.

40. **⚠️ Verify page-offset mapping per chapter.** Run a Python script to confirm file-name match after bulk translation.

41. **execute_code is unreliable for writing translated content.** Use `write_file` — it handles arbitrary content reliably. Use `execute_code` only for PDF rendering, extraction, offset verification, and auditing.

42. **⚠️ User-directed hybrid fast-then-verify workflow.** When user requests efficiency: batch-render, pymupdf extract, bulk translate, visual spot-check 3-5 pages, fix mismatches.

### Images & Figures

43. **Skipping image extraction or verification.** Every page must be checked for images. Always verify against original with `vision_analyze`.

44. **⚠️ `get_images()` produces decorative tiles, NOT book figures.** Real book figures are vector graphics. Detection: any extracted image under 20KB or under 400×400px is decorative.

45. **Image clip extraction uses proportional coordinates, not pixel guesses.** Use vision to find bounding box as fractions, compute `fitz.Rect`. Expect 2-3 refinement rounds.

46. **⚠️ Figure cropping MUST produce equal margins on all four sides.** Full 8-step pipeline: locate → render → vision proportion → text-block refine → clip → ink detect + ImageOps.expand → exclude rules → vision verify. Target 50px margins.

47. **FIG↔PAGE cross-reference integrity.** Every FIG file must be referenced in exactly one PAGE file. Every `![[` reference must point to an existing FIG file.

48. **FIG files must be embedded in the correct PAGE that contains the figure.** Cross-reference every "图X.Y" mention against FIG files.

49. **⚠️ Full-page renders saved as FIG files — skipped the 8-step cropping pipeline entirely.** Detection: `PIL.Image.open(f).size[1] > 800` = full page, not cropped. All 11 Ch4 figures had this issue. Prevention: after any batch extraction, run the dimension check and route through 8-step pipeline.

### Auditing & Verification

50. **⚠️ After bulk translation, run a systematic content audit on ALL pages.** Two-part audit: (A) scan for truncation signatures, (B) scan prev/next links. Word-count ratio can flag potential truncation but has false negatives on dialogue pages (ratio looks healthy even with 40% missing) and false positives on code-heavy pages. The tail-alignment check is the real discriminator: ratio + tail mismatch = confirmed truncation; ratio alone = investigate. Reusable audit script in `references/bulk-audit-script.md`.

## Verification Checklist

- [ ] All `[[wikilink]]` targets exist as files
- [ ] Page sequence continuous (no gaps)
- [ ] prev/next links bidirectional
- [ ] Book file TOC covers all top-level sections
- [ ] Every TOC entry has both `-[[SECTION]]` and `-[[PAGE起始页]]`
- [ ] Every section has a summary; no page has a summary
- [ ] All FIG files verified against original PDF pages
- [ ] Every PAGE file translated from pymupdf text
- [ ] Step 3 pre-translate gate passed for every page
- [ ] Step 3.5 visual layout verified for every page
- [ ] Cross-page continuity: PAGE N flows into PAGE N+1
- [ ] Zero translator artifacts: `(接上页)`, `（续）`, `*待续*`
- [ ] FIG↔PAGE cross-reference integrity
- [ ] FIG cropping: all files under 800px height, equal margins
- [ ] No printer's marks in any PAGE content
- [ ] Systematic content audit passed (see pitfall #50)
- [ ] Vault synced after all writes

## Related References

- `references/ch3-before-after-examples.md` — 9 real audit cases from IMPLDDD Ch3: layout reversal, extra figures, blockquote drift, composite figure duplication, cross-chapter refs, cross-page content, FIG cropping failures. Each case has BEFORE/AFTER code blocks, detection steps, and pitfall cross-references.
- `references/page-file-concrete-example.md` — PAGE/SECTION/TOC template examples
- `references/pdf-page-mapping.md` — PDF absolute to printed page mapping
- `references/batch-rendering.md` — Batch page rendering pattern
- `references/visual-translation-workflow.md` — Hybrid pymupdf + vision workflow
- `references/image-extraction-failures.md` — Image extraction failure modes
- `references/parallel-translation-pattern.md` — Parallel subagent delegation
- `references/section-filling.md` — Batch-fill SECTION summaries
- `references/parallel-vision-figure-scan.md` — Parallel figure scanning
- `references/equal-margin-figure-crop.md` — Equal-margin figure cropping
- `references/search-files-roman-numeral.md` — Roman numeral filename workaround
- `references/page-layout-audit-methodology.md` — Multi-page layout audit
- `references/bulk-audit-script.md` — Reusable word-count + tail-alignment audit script
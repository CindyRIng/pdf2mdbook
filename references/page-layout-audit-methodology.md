# Page Layout Audit Methodology

Systematic method for auditing a batch of MDBOOK PAGE files against the original PDF — catching visual-order errors, blockquote boundary drift, and extra/missing figures.

Proven on IMPLDDD Chapter 3 (PAGE87-99, 13 pages). Found 3 errors across 13 pages that text-only review would miss.

## Why This Is Necessary

PDF text extraction (`page.get_text("text")`) returns content in internal byte-stream order, NOT visual reading order. This is a PDF format inherent property — not a bug in any tool. It affects ALL PDFs regardless of language, publisher, or creation tool. Three error classes are invisible without visual verification:

| Error type | Cannot be detected by | Example |
|---|---|---|
| Figure position wrong | Text extraction alone | FIG3-5 at page bottom instead of top (IMPLDDD PAGE99) |
| Extra/missing figures | Text extraction alone | FIG3-6 present on PAGE99, actually on PAGE100 |
| Blockquote boundary drift | Text extraction alone | Plain body text wrapped in `>` (IMPLDDD PAGE97-98) |

## Audit Workflow

### Phase 1: Batch Render Original Pages

```python
import fitz, os

pdf_path = "path/to/book.pdf"
outdir = "/tmp/audit_pages"
os.makedirs(outdir, exist_ok=True)

doc = fitz.open(pdf_path)
offset = 42  # PDF page - offset = printed page (verify per book)

for printed_page in range(start, end + 1):
    pdf_page = printed_page + offset
    page = doc[pdf_page]
    pix = page.get_pixmap(dpi=150)
    pix.save(f"{outdir}/page_{printed_page:03d}.png")

doc.close()
```

### Phase 2: Per-Page Visual Analysis

For each page, call `vision_analyze` with the rendered PNG and these exact questions:

1. **Layout order:** "Describe visual layout top-to-bottom. What appears at the very top? In the middle? At the bottom? Exact order of ALL elements (headings, body text, figures, callout boxes, sidebars, horizontal rules)."

2. **Figure inventory:** "List every numbered figure on this page. Where is each positioned? What is the exact figure number (e.g., Figure 3.4, Figure 3.5)?"

3. **Callout boundaries:** "Which text segments are visually inside callout boxes (gray background, left vertical bar, horizontal rules above/below) vs plain body text?"

### Phase 3: Compare Against MDBOOK

For each page, compare vision_analyze output against the MDBOOK PAGE file:

1. **Element order:** Is the first element in MDBOOK the same as what appears at the TOP of the original?
2. **Figure count:** Number of `![[../FIG/` in MDBOOK = number of figures on original page?
3. **Figure position:** Is each `![[` at the same relative position as in the original?
4. **Blockquote boundaries:** For every `>` in MDBOOK, is that text inside a callout box in the original?
5. **Missing callouts:** Are there callout boxes in the original that are NOT marked `>` in MDBOOK?

### Phase 4: Summarize Findings

Create a matrix:

| Page | Status | Figures | Callout Issues | Notes |
|------|--------|---------|----------------|-------|
| 87 | ✅ | 0 | — | Pure text page |
| 88 | ✅ | FIG3-1 at top | — | Figure-first layout |
| ... | ... | ... | ... | ... |
| 97 | ⚠️ | FIG3-4 mid | First section wrongly `>` | Body text + figure in callout |
| 99 | ❌ | FIG3-5 should be TOP | FIG3-6 doesn't belong | Double error |

## Error Patterns Found (IMPLDDD Chapter 3)

### Pattern 1: Blockquote Boundary Drift

When a page mixes callout and plain text, `>` markers leak into adjacent paragraphs. Root cause: pymupdf extraction loses visual formatting cues (gray background, vertical bars, horizontal rules). Fix: vision_analyze to identify which paragraphs are inside callout boxes.

### Pattern 2: Figure Position Reversal

Figures rendered at the TOP of the original page appear at the BOTTOM in MDBOOK. Root cause: PDF internal byte-stream order places figure data after text data. Fix: vision_analyze to determine visual order, place `![[FIG]]` at matching position.

### Pattern 3: Extra/Missing Figures

A figure from an adjacent page incorrectly embedded (FIG3-6 on PAGE99, actually on PAGE100). Root cause: bulk figure-embedding scripts attach figures based on text reference proximity, not visual page boundaries. Fix: for every `![[FIG-X-Y]]`, render the original page and confirm the figure is physically present there.

## Scaling: Full-Book Audit

For a full book (600+ pages), batch in groups of 10-13 pages:
1. Batch-render pages in groups
2. Run vision_analyze on each group (parallelizable with delegate_task)
3. Compare against MDBOOK files
4. Fix errors per group before moving to the next

Not every page needs full visual verification — pages with no figures AND no callout boxes AND uniform body text are safe to skip the Tier A visual layout audit. HOWEVER, these text-only pages still need the Tier B lightweight content-integrity check: compare the last ~50 chars of the MDBOOK page against the last ~50 chars of the PDF page (after stripping page numbers and headers). If they don't align, the page was truncated during extraction or translation. See pitfall #36e. Pages with ANY of the following MUST be verified with full Tier A: figures, callout boxes, sidebars, mixed formatting, unusual layout.

# MDBOOK Content Audit

Systematic scan scripts for detecting truncation and broken navigation after bulk translation.

## When to Run

- After bulk translation of any chapter (especially if any pages were ever vision-translated)
- After deleting or reordering PAGE files
- Before declaring translation of a section "complete"
- When user reports "this page seems incomplete"

## Word-Count Ratio Scan (Quick Heuristic)

Before diving into content analysis, run a quick ratio check — this catches severe truncation instantly:

```python
import fitz, os

doc = fitz.open("book.pdf")
BASE = "CONTENT/"
# Map printed pages to PDF pages (book-specific offset)
# Example: frontmatter offset = 1, body offset = 42

for printed, pdf_page in page_map.items():
    pdf_text = doc[pdf_page].get_text('text')
    pdf_words = len(pdf_text.split())
    
    fname = f'{BASE}/BOOKNAME-PAGE{printed}.md'
    with open(fname) as f:
        md_text = f.read()
    chinese_chars = sum(1 for c in md_text if '\u4e00' <= c <= '\u9fff')
    
    # Chinese translation typically 0.5-0.7 chars per English word
    # Ratio < 0.3 = likely truncated (>50% missing)
    ratio = chinese_chars / max(pdf_words, 1)
    if ratio < 0.3:
        print(f"PAGE{printed}: SEVERE TRUNCATION (ratio {ratio:.2f})")
    elif ratio < 0.5:
        print(f"PAGE{printed}: POSSIBLE TRUNCATION (ratio {ratio:.2f})")
```

**Production finding (IMPLDDD):** PAGEXXVIII had ratio 0.80 (390ch / 489w) — appeared OK but was actually missing 40% of its paragraph count. PAGEXXXI had ratio 0.77 (288ch / 374w) — missing 75%. The ratio heuristic flags severe cases well; for borderline cases, cross-reference with paragraph counts.

## Content Truncation Scan

Detects pages whose last content line shows signs of truncation:

- Ends with `--` (mid-sentence dash cutoff)
- Is very short (<15 chars) and not a legitimate marker (`*续*`, `（续）`, ` ``` `, `## （空白页）`)
- Ends abruptly (e.g., `集成限` when the full title is `集成限界上下文`)

```python
from hermes_tools import terminal
import re

BASE = "/path/to/MDBOOK/CONTENT"
result = terminal(f"ls {BASE}/ | grep '^BOOKNAME-PAGE' | sort -V")
files = [f.strip() for f in result["output"].split("\n") if f.strip()]

for fname in files:
    page = fname.replace("BOOKNAME-", "").replace(".md", "")
    r = terminal(f"cat {BASE}/{fname}")
    content = r["output"]
    lines = content.split("\n")
    
    # Find last non-empty content line (after "# 内容")
    content_lines = []
    in_content = False
    for line in lines:
        if line.startswith("# 内容"):
            in_content = True
            continue
        if in_content:
            s = line.strip()
            # Skip image embeds, code fences, and empty lines
            if s and not s.startswith("![[") and not s.startswith("```"):
                content_lines.append(s)
    
    if not content_lines:
        print(f"{page}: EMPTY (no content lines)")
        continue
    
    last = content_lines[-1]
    
    # Check for truncation patterns
    if len(last) < 15:
        # Known-good short endings (NOTE: *续* and （续） are forbidden per pitfall #22)
        if last in ("```", "}", "|") or last.startswith("##"):
            continue
        print(f"{page}: SHORT ({len(last)}c): '{last}'")
    elif re.search(r'[^\s]--\s*$', last):
        print(f"{page}: DASH: '{last[-80:]}'")
    elif len(last) < 50 and not re.search(r'[。！？]$', last):
        print(f"{page}: SUSPICIOUS: '{last}'")

# Also flag pages whose content ends without a sentence terminator
# where the line is long enough to be prose
for fname in files:
    r = terminal(f"cat {BASE}/{fname}")
    content = r["output"]
    # ... check last prose line for Chinese sentence endings
```

## Navigation Integrity Scan

After deleting pages or restructuring, check for prev/next links pointing to non-existent pages:

```python
from hermes_tools import terminal

BASE = "/path/to/MDBOOK/CONTENT"

# Build list of existing pages
result = terminal(f"ls {BASE}/ | grep '^BOOKNAME-PAGE' | sort -V")
existing_files = set(f.strip().replace("BOOKNAME-", "").replace(".md", "") for f in result["output"].split("\n") if f.strip())

# Scan nav links in every page
for fname in existing_files:
    r = terminal(f"head -6 {BASE}/BOOKNAME-{fname}.md")
    content = r["output"]
    # Check prev and next links
    for line in content.split("\n"):
        if "上一页" in line or "下一页" in line:
            # Extract page name from [[BOOKNAME-PAGEXYZ]]
            match = re.search(r'\[\[BOOKNAME-(PAGE\w+)\]\]', line)
            if match:
                target = match.group(1)
                if target not in existing_files:
                    print(f"{fname}: BROKEN LINK → {target}")
```

## Fix Workflow for Truncation Hits

1. **Locate the PDF:** Search `Resource/PDF/` and the 18088 reverse tunnel
   ```bash
   find ~/obsidian/Resource/PDF -name '*BookTitle*.pdf'
   curl -s http://127.0.0.1:18088/ | grep -i bookname
   ```

2. **Determine page offset:** Render the first few pages to map PDF page → printed book page
   ```python
   import fitz
   doc = fitz.open("book.pdf")
   for i in range(min(30, doc.page_count)):
       text = doc[i].get_text("text")
       first_line = text.strip().split("\n")[0]
       if first_line.isdigit() or first_line.lower() in ["xvii", "xviii"]:
           print(f"PDF page {i} → printed page {first_line}")
   ```

3. **Extract and retranslate:**
   ```python
   for printed_page in affected_pages:
       pdf_page = printed_page + offset  # verify per book
       text = doc[pdf_page].get_text("text")
       # Translate faithfully to Chinese
       # Write corrected PAGE file
   ```

4. **Verify cross-page continuity:**
   ```bash
   # Check that corrected page's ending flows into next page's beginning
   tail -2 CONTENT/BOOKNAME-PAGEXXIV.md
   head -12 CONTENT/BOOKNAME-PAGEXXV.md
   ```

## Production Example: IMPLDDD Audit

**Round 1 (Ch1 audit):** Scanned 68 PAGE files. Found 6 content-truncated pages in the Preface (XXIV-XXX) — all from an earlier vision-only translation pass. All fixed by re-extracting from PDF with pymupdf and retranslating.

**Round 2 (Frontmatter user review):** User manually reviewed the Preface section and found a cascade of truncations: PAGEXX (missing 80% — cut at "着陆并"), PAGEXXI (missing 60% — cut at "在一个"), PAGEXXII (missing 50% — cut at "起飞"), PAGEXXIII (duplicate heading + missing flow), PAGEXXVIII (missing 40% — relational DB paragraphs), PAGEXXXI (missing 75% — reviewers/illustrations/parents paragraphs). **Critical pattern: when user flags one truncated page, immediately scan the entire containing section (±2 pages). Truncation never strikes in isolation.**

**Round 3 (GUIDE TO THIS BOOK):** Word-count ratio scan revealed PAGEXXXVI, PAGEXXXVIII, PAGEXXXIX, PAGEXL all severely truncated (ratios 0.3-0.6). These pages are image-heavy but still had significant prose missing — the Aggregates explanation, Modules paragraph, closing remarks, and architecture prioritization paragraph were all lost.

**Root cause:** These pages were translated via vision_analyze earlier in the project, before the pymupdf-primary workflow was established. The systematic audit + user review combination is the safety net.

**Fix workflow used in production:**
1. Extract full page text from PDF: `doc[pdf_page].get_text("text")`
2. Translate faithfully to Chinese (preserve all paragraphs, bullet lists, headings)
3. Write corrected PAGE file
4. Verify cross-page flow: does corrected page's ending flow into next page's beginning?
5. Expand scan ±2 pages — fixing one almost always reveals neighbors are also damaged

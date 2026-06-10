# Parallel Subagent Translation Pattern

Proven across 640-page IMPLDDD conversion (68 frontmatter + 572 body pages). Uses 3 parallel subagents per round.

## When to Use

- 50+ pages remaining to translate
- Book already has TEXT extracted via pymupdf into `/tmp/` files
- PAGE file skeletons already created with `<待补充>` placeholders

## Workflow

### Phase 1: Source Extraction (one-time, parent agent)

```python
import fitz, os

PDF = "path/to/book.pdf"
doc = fitz.open(PDF)
OUT = "/tmp/bookname_pages"
os.makedirs(OUT, exist_ok=True)

OFFSET = 42  # PDF page = printed page + offset

for printed in range(start_page, end_page + 1):
    pdf_page = printed + OFFSET
    text = doc[pdf_page].get_text("text")
    with open(f"{OUT}/page{printed}_en.txt", "w") as f:
        f.write(text)

doc.close()
```

### Phase 2: Skeleton Creation (one-time, parent agent)

```python
BASE = "vault_path/Resource/MDBOOK/BOOKS/MDBOOK<YYMMDD>-<BOOK>/CONTENT"
BOOK = "MDBOOK<YYMMDD>-<BOOK>"

for page_num in range(start, end + 1):
    fname = f"{BASE}/{BOOK}-PAGE{page_num}.md"
    if not os.path.exists(fname):
        content = f"""# 链接

- **目录：** [[{BOOK}]]
- **上一页：** [[{BOOK}-PAGE{page_num - 1}]]
- **下一页：** [[{BOOK}-PAGE{page_num + 1}]]

# 内容

<待补充>
"""
        with open(fname, "w") as f:
            f.write(content)
```

### Phase 3: Parallel Translation (rounds of 3 subagents)

**Batch size rules:**
- Body-text pages (dense prose): **25-40 pages per agent**
- Code-heavy pages (Chapter 5, 12, Appendix A): **40-50 pages per agent**
- INDEX/backmatter pages: **handle directly by parent, do not delegate**

**Per-agent context template:**
```
You are translating pages {start}-{end} of 'Book Title' by Author.
Chapter info: {chapter_summary}.

SOURCE FILES: English text at /tmp/bookname_pages/page{N}_en.txt (N={start} to {end})
TARGET FILES: Write to {base_path}/MDBOOK{date}-{book}-PAGE{N}.md (N={start} to {end})

Each target file has a skeleton with <待补充>. Replace <待补充> with Chinese translation under # 内容.

FILE FORMAT:
# 链接
- **目录：** [[MDBOOK{date}-{book}]]
- **上一页：** [[MDBOOK{date}-{book}-PAGE{N-1}]]
- **下一页：** [[MDBOOK{date}-{book}-PAGE{N+1}]]

# 内容
Chinese translation...

TRANSLATION RULES:
- Translate ALL text faithfully - every paragraph, bullet, callout box
- Code blocks (Java/C#/SQL) stay English in ``` fences
- Proper names stay original; DDD terms stay English
- Strip repeated page headers
- Preserve printed page number as first content line

Use write_file for each PAGE. Process ALL {count} pages.
```

### Phase 4: Timeout Recovery

Subagents with 50+ pages frequently hit the 600s timeout. Recovery pattern:

1. **Scan for remaining work:**
```python
pending = []
for pg in range(start, end + 1):
    with open(f"{BASE}/{BOOK}-PAGE{pg}.md") as f:
        if "<待补充>" in f.read():
            pending.append(pg)
```

2. **Re-delegate only the untranslated sub-range.** Typically 10-25 pages remain — a perfect size for a single agent to complete within the time limit.

3. **If <25 pages remain, parent translates directly** using `write_file` calls — faster than another subagent round.

### Phase 5: Cross-Page Verification

After all pages are translated:
1. Spot-check 3-5 pages per chapter for visual quality
2. Run content audit scan (see `references/content-audit.md`)
3. Run navigation integrity scan
4. Verify cross-page continuity at chapter boundaries

## Real Example: IMPLDDD 640-page Conversion

| Round | Agents | Pages | Result |
|-------|--------|-------|--------|
| 1 | 3 agents | 43-170 (Ch2-4, 128p) | Agent A timeout (Ch2, 44p), B+C completed |
| 1 retry | 2 agents | 43-86 split (Ch2, 44p) | Both completed (A found 43-65 already done) |
| 2 | 3 agents | 171-346 (Ch5-9, 176p) | Agent A+B found done, C timeout |
| 2 retry | 3 agents | 219-295 (Ch6-8, 77p) | All completed (most already done) |
| 3 | 3 agents | 296-538 (Ch8-14, 243p) | Agent A completed, B+C timeout |
| 3 retry | 3 agents | 347-510 (Ch10-13, 164p) | All completed (most already done) |
| 4 | 2 agents | 511-614 (Ch14+AppA, 104p) | Both timeout |
| 4 retry | parent | 564-576, 603-614 (25p) | Direct write_file |

**Key insight:** Timeouts leave partial work on disk. The "retry" rounds consistently found 70-90% of pages already done — each successive round had dramatically less work.

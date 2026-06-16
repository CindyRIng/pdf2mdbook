# Bulk Audit Script — Reusable Pattern

Run this Python script against any MDBOOK page range to detect truncation, running headers, bare page numbers, and other common issues.

```python
import fitz
import os
import re

PDF = "<path-to-pdf>"
MDBOOK_DIR = "<path-to-content-dir>"
OFFSET = 42  # book_page = pdf_page - OFFSET; verify per book

doc = fitz.open(PDF)

for book_page in range(START, END + 1):
    pdf_page = book_page + OFFSET
    if pdf_page >= doc.page_count:
        break
    
    # PDF extraction
    page = doc[pdf_page]
    pdf_text = page.get_text("text")
    pdf_words = len(pdf_text.split())
    
    # MDBOOK file
    md_path = os.path.join(MDBOOK_DIR, f"<BOOK>-PAGE{book_page}.md")
    if not os.path.exists(md_path):
        print(f"PAGE{book_page}: FILE_MISSING")
        continue
    
    with open(md_path, 'r') as f:
        md_content = f.read()
    
    content = md_content.split('# 内容')[1] if '# 内容' in md_content else md_content
    md_chars = sum(1 for c in content if '\u4e00' <= c <= '\u9fff')
    
    ratio = pdf_words / max(md_chars, 1)
    
    # Tail alignment (primary detection — runs unconditionally)
    pdf_lines = [l.strip() for l in pdf_text.split('\n') if l.strip()]
    pdf_tail = ' '.join(pdf_lines[-3:])[-80:]
    
    md_lines = [l.strip() for l in content.split('\n') if l.strip() and not l.startswith('![[../FIG')]
    md_tail = ' '.join(md_lines[-3:])[-80:]
    
    issues = []
    
    # 1. Ratio flagging (investigation signal only, NOT proof)
    if ratio > 2.0:
        issues.append(f'RATIO_SEVERE({ratio:.1f})')
    elif ratio > 1.5:
        issues.append(f'RATIO_HIGH({ratio:.1f})')
    
    # 2. Running headers: ONLY flag ALL-CAPS English patterns
    first_line = content.split('\n')[0].strip() if content else ''
    if re.match(r'Chapter \d+ [A-Z ,]+', first_line):
        issues.append('RUNNING_HEADER')
    
    # 3. Bare page numbers: ONLY flag digits >= 10 (avoid console output false positives)
    for line in content.split('\n'):
        stripped = line.strip()
        if stripped.isdigit() and int(stripped) >= 10:
            issues.append('BARE_PAGE_NUM')
            break
    
    # 4. Translator artifacts
    if '接上页' in content:
        issues.append('ARTIFACT')
    
    # Print results
    status = '✓' if not issues else '⚠ ' + ','.join(issues)
    print(f"PAGE{book_page:>3} | W={pdf_words:>4} C={md_chars:>4} R={ratio:.2f} | {status}")
    
    # For flagged pages, show tails for manual comparison
    if issues:
        print(f"  PDF tail: ...{pdf_tail[-60:]}")
        print(f"  MD  tail: ...{md_tail[-60:]}")
        print()

doc.close()
```

## Usage

1. Adjust `PDF`, `MDBOOK_DIR`, `OFFSET`, `START`, `END`, and `<BOOK>` for your project
2. Run: `~/venv/bin/python3 audit.py`
3. **Triage every flag manually** — ratio alone is NOT proof of truncation:
   - Ratio + tail mismatch → confirmed truncation
   - Ratio alone → investigate; may be code-heavy page (false positive) or dialogue page (false negative)
4. Pages flagged with ratio < 1.5 but suspicious tail alignment → manually re-check

## Known False Positives

- Code-heavy pages (Java, SQL, XML) inflate PDF word count
- Short pages (< 100 PDF words) have noisy ratio
- Body text containing "架构" is NOT a running header
- Single-digit console output ("3") is NOT a bare page number

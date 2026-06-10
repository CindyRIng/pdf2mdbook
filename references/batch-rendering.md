# Batch Page Rendering for MDBOOK

Use this pattern to render 10-40 PDF pages in a single Python pass. Run as a single `terminal` call before starting vision or pymupdf extraction.

```python
import fitz

pdf_path = "<path to PDF>"
doc = fitz.open(pdf_path)

# Render a range of book pages to /tmp/
# book_page → PDF page: offset = 42 for IMPLDDD main body; adjust per book
offset = 42
for book_page in range(start, end + 1):
    pdf_page = book_page + offset
    page = doc[pdf_page]
    pix = page.get_pixmap(dpi=200)
    out = f"/tmp/impl_page{book_page}.png"
    pix.save(out)
    print(f"Saved PAGE{book_page}")

doc.close()
print(f"DONE: rendered {end - start + 1} pages")
```

**Run it:**
```
python3 << 'PYEOF'
... script ...
PYEOF
```

**Tips:**
- Render in chunks of 10-15 pages to keep vision_analyze calls manageable
- Use consistent naming: `/tmp/impl_page{N}.png` — easy to reference
- 200 DPI is the sweet spot: readable text, reasonable file size
- Always render BEFORE starting translation — this decouples I/O from translation work

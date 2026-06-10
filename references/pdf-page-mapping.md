# PDF Page Number Mapping

## IMPLDDD Book

PDF: `Implementing Domain-Driven Design (Vaughn Vernon)`

| Book page | PDF page | Offset |
|-----------|----------|--------|
| I-IX (frontmatter cover) | 0-8 | PDF page number |
| IX (Contents start) | 9 | PDF page number |
| XVII (Foreword) | 17 | PDF page number |
| XIX (Preface) | 19 | PDF page number |
| 1 (Chapter 1) | 43 | PDF - 42 |
| 2 (Chapter 1) | 44 | PDF - 42 |
| N (main body) | N + 42 | PDF - 42 |

**Formula:**
- Frontmatter (Roman numerals): `book_page = PDF_page` (same number)
- Main body (Arabic numerals): `book_page = PDF_page - 42`

## General approach for any book

1. Open PDF, scan first few pages
2. Find the first Roman numeral page and its PDF index
3. Find the first Arabic numeral page (usually chapter 1, page 1) and its PDF index
4. Verify: the gap between Roman and Arabic pages should be consistent
5. Document the mapping and use it for ALL `页码范围` and PAGE file names

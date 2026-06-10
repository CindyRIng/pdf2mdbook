# search_files Roman Numeral Pitfall

## Problem

`search_files` regex patterns fail to match filenames containing Roman numerals (PAGEXVII, PAGEXX, PAGEXIX, etc.) when using standard grep-like patterns like `PAGEXV`, `PAGEXX`, or `PAGEX`.

**Observed:** In IMPLDDD MDBOOK CONTENT directory, `search_files(pattern="PAGEXV", target="files")` returned 0 results, but `ls .../CONTENT/MDBOOK*-PAGEX*` showed all the Roman-numeral PAGE files actually existed.

## Root Cause

Unknown — likely a regex escaping or character-class mismatch in `search_files`'s underlying ripgrep invocation when Roman numeral characters (X, V, I, L) appear in specific sequences.

## Workaround

Use `terminal` with `ls` to find files with Roman numeral patterns:

```bash
ls /path/to/CONTENT/MDBOOK*-PAGEX* 2>/dev/null
ls /path/to/CONTENT/MDBOOK*-PAGEXX* 2>/dev/null
ls /path/to/CONTENT/MDBOOK*-PAGEXXX* 2>/dev/null
```

The `ls` glob always works because it's shell-level filename matching, not regex.

## When This Matters

- MDBOOK frontmatter pages use Roman numerals (PAGEXVII, PAGEXVIII, PAGEXXIX, etc.)
- Verifying which SECTION pages exist before fixing missing links
- Auditing PAGE↔SECTION cross-references

## TL;DR

If `search_files` returns 0 for a known-existing file pattern, fall back to `ls` in terminal before concluding the files don't exist.

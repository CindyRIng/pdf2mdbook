# Section Filling Script Pattern

Used in IMPLDDD to fill 231 SECTION files (181 leaf + 50 non-leaf) with summaries, page links, and sub-chapter links.

## Part 1: Leaf Sections (summaries + page links)

```python
from hermes_tools import terminal
import re, os

CONTENT = "/path/to/MDBOOK/CONTENT"
BOOK = "MDBOOK260503-IMPLDDD"

# Read all SECTION files
result = terminal(f"ls {CONTENT}/ | grep '^{BOOK}-SECTION' | sort -V")
sec_files = [f.strip() for f in result["output"].split("\n") if f.strip()]

sections = {}
for fname in sec_files:
    sec_id = fname.replace(f"{BOOK}-SECTION", "").replace(".md", "")
    r = terminal(f"cat {CONTENT}/{fname}")
    content = r["output"]
    
    name_match = re.search(r'章节全称:(.+)', content)
    range_match = re.search(r'页码范围:(.+)', content)
    
    name = name_match.group(1).strip() if name_match else ""
    page_range = range_match.group(1).strip() if range_match else ""
    
    start_page = end_page = None
    if " - " in page_range:
        parts = page_range.split(" - ")
        try:
            start_page = int(parts[0].strip())
            end_page = int(parts[1].strip())
        except:
            pass
    
    sections[sec_id] = {
        "id": sec_id, "name": name, "start": start_page, "end": end_page,
        "file": fname, "content": content
    }

# Find children to distinguish leaf vs non-leaf
def sort_key(s):
    parts = s.replace("A", "15").split(".")
    return tuple(int(p) if p.isdigit() else p for p in parts)

sorted_ids = sorted(sections.keys(), key=sort_key)
children = {}
for sid in sorted_ids:
    parts = sid.split(".")
    if len(parts) > 1:
        parent = ".".join(parts[:-1])
        if parent not in children:
            children[parent] = []
        children[parent].append(sid)

# Summary library — write short Chinese summaries for key sections
# For sections without explicit summaries, generate from name pattern
summaries = {
    "2.1.1": "通过SaaSOvation案例展示如何在实际项目中识别子域和限界上下文...",
    "2.6": "回顾第2章的核心要点...",
    # ... one entry per leaf section
}

# Fill all leaf sections
for sid in sorted_ids:
    sec = sections[sid]
    is_leaf = sid not in children and "<待补充>" in sec["content"]
    if not is_leaf:
        continue
    
    summary = summaries.get(sid, f"本节围绕'{sec['name']}'展开。")
    
    # Generate page links from range
    page_links = ""
    if sec["start"] and sec["end"]:
        pages = []
        for p in range(sec["start"], sec["end"] + 1):
            pages.append(f"- [[{BOOK}-PAGE{p}]]")
        page_links = "\n".join(pages)
    
    new_content = sec["content"].replace("<待补充>", summary, 1)
    if page_links:
        new_content = new_content.replace("<待补充>", page_links, 1)
    
    with open(f"{CONTENT}/{sec['file']}", "w") as f:
        f.write(new_content)
```

## Part 2: Non-Leaf Sections (summaries + sub-chapter links)

```python
parent_summaries = {
    "2": "第2章介绍DDD战略设计的核心工具...",
    "2.1": "通过全局概览介绍子域与限界上下文的核心概念...",
    # ... one entry per non-leaf section
}

for sid in sorted_ids:
    sec = sections[sid]
    is_parent = sid in children and "<待补充>" in sec["content"]
    if not is_parent:
        continue
    
    summary = parent_summaries.get(sid, f"本节是{sid}的父级导航节点。")
    
    # Generate sub-chapter links
    sub_links = ""
    if sid in children:
        sub_ids = sorted(children[sid], key=sort_key)
        links = [f"- [[{BOOK}-SECTION{s}]]" for s in sub_ids]
        sub_links = "\n".join(links)
    
    new_content = sec["content"].replace("<待补充>", summary, 1)
    if sub_links:
        new_content = new_content.replace("<待补充>", sub_links, 1)
    
    with open(f"{CONTENT}/{sec['file']}", "w") as f:
        f.write(new_content)
```

## Pitfalls

1. **Empty page ranges (0p entries)**: Some SECTION files have `页码范围: 136 - 135` (end < start). Skip page-link generation for these — they represent virtual/conceptual sub-sections with no physical pages.
2. **Section IDs containing "A"**: Appendices use "A" as chapter number. The sort key must map `"A"` to `"15"` to sort correctly after Chapter 14.
3. **Frontmatter sections (0.x)**: These have Roman numeral page ranges (XIX - XXVIII). They need special handling — or can be filled manually since there are only ~20 of them.
4. **Summary quality**: For 181 leaf sections, writing individual summaries from page content is impractical. Use descriptive section names from the book's TOC as the primary signal, with generic fallbacks for obscure sub-sections.

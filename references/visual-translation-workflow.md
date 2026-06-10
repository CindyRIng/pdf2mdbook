## Visual Translation Workflow — Updated with Truncation Pitfall

### The Problem

Both text extraction methods have complementary reliability gaps:

| Method | Silent failure mode | What gets lost |
|---|---|---|
| pymupdf `get_text("text")` | Reorders/drops content | Callout boxes, bullet lists, sidebars, code formatting |
| `vision_analyze` transcription | **Silent token-limit truncation** | Bottom 20-40% of dense pages |

**Evidence from IMPLDDD Ch1:** Every page translated via single-pass vision was missing bottom content. PAGE1 had 1 of 7 roadmap bullets. The truncation is silent — outputs look complete but aren't. Vision even fails on half-page reads for very dense pages (PAGE2 bottom half was still truncated).

### Mandatory Hybrid Workflow (per page)

```
1. pymupdf extract → complete text (no token limit)
2. vision_analyze verify bottom → check for missed callouts/boxes and truncation
3. Translate from pymupdf text (faithful, every paragraph)
4. Write PAGE file
5. Visual spot-check every 5-7 pages for quality
```

### Two-Pass Vision (when vision-only is unavoidable)

```
1. Pass 1: vision_analyze(question="Read ONLY the TOP HALF...")
2. Pass 2: vision_analyze(question="Read ONLY the BOTTOM HALF. Start where top left off...")
3. Merge both halves → translate
4. Pass 3: vision_analyze(question="Read the LAST 5 lines. Are they complete?")
```

**Even two-pass can fail on dense pages.** If the bottom half is still truncated, do a third pass for the very bottom. Verify paragraph counts match between source and output.

### Parallel Subagent Delegation Pattern

For bulk translation (10+ pages), use `delegate_task` with 3 parallel subagents:

```
Batch 1: pages X-Y (first third)     Batch 2: pages Y+1-Z (second third)     Batch 3: pages Z+1-end (final third)
```

Each subagent gets: exact file paths, pymupdf source text (saved to /tmp/), PAGE format template, translation instructions. Toolsets: `["terminal","file"]`.

**After subagents complete, the parent MUST:**
1. Spot-check boundary pages visually (vision_analyze verify bottom) — pages near splits often have content overflow
2. Fix boundary overlaps — trim the earlier page ending, prepend missing opening to the later page (e.g., PAGE35's last bullet spilled into PAGE36)
3. Verify prev/next links — check no `(待补充)` remains
4. Sync the vault

### Image Extraction Quality Failures

The vision-clip approach requires **mandatory post-extraction verification**. Common failures:

| Failure | Symptom | Fix |
|---|---|---|
| Text bleeding | Body paragraphs included in image | Tighten crop |
| Over-crop | Figure edges cut off | Loosen crop |
| Blank extraction | Completely white image | Wrong clip region — re-locate |
| Wrong content | Caption + text but no diagram | Figure on different page/region |

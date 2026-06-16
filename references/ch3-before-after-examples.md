# Chapter 3 Before/After Examples — IMPLDDD

Real audit cases extracted from IMPLDDD Ch3 (PAGE87–112), mapping each error to its detection method, fix, and skill pitfall.

---

## 1. Layout Reversal — Figure at Wrong Position

**Page:** PAGE99 | **Pitfall:** #14 (Extraction order ≠ visual order)

**Error:** FIG3-5 placed at page BOTTOM in MDBOOK (text first, then figure). Original book page: FIG3-5 at TOP (40% of page), discussion text MIDDLE, Cowboy Logic sidebar BOTTOM.

**Detection:**
```python
# Render original PDF page
pix = page.get_pixmap(dpi=150)
pix.save("/tmp/page_099.png")
# Then: vision_analyze("Describe visual layout top-to-bottom")
```

**Fix — BEFORE (wrong):**
```markdown
## 第3章 上下文映射

我们的新核心域，我们会期望它位于图的顶部或中心...

![[../FIG/MDBOOK260503-IMPLDDD-FIG-3-5.png]]

***图3.5*** *当前核心域用粗边界标记...*
```

**Fix — AFTER (correct):**
```markdown
## 第3章 上下文映射

![[../FIG/MDBOOK260503-IMPLDDD-FIG-3-5.png]]

***图3.5*** *当前核心域用粗边界标记并标有集成点。CollabOvation支撑子域和IdOvation通用子域位于上游。*

我们的新核心域，我们会期望它位于图的顶部或中心...

> **牛仔逻辑**
```

**Prevention:** Step 3.5 Tier A mandatory. Never trust pymupdf text extraction order for page layout.

---

## 2. Extra Figure — Wrong Page

**Pages:** PAGE99 (FIG3-6), PAGE101 (FIG3-7), PAGE111 (FIG3-8) | **Pitfall:** #17

**Error:** FIG files embedded on pages where the original PDF has NO figure. FIG3-6 belongs on PAGE100, FIG3-7 belongs on PAGE102, FIG3-8 belongs on PAGE105.

**Detection:** Render original PDF page → confirm each `![[../FIG/` reference corresponds to an actual figure on that page.

**Fix — BEFORE (PAGE101, wrong):**
```markdown
> 问题仍然存在：创建上下文映射就这些了吗？可能吧。

![[../FIG/MDBOOK260503-IMPLDDD-FIG-3-7.png]]

***图3.7*** *一个逻辑翻译映射...*
```

**Fix — AFTER (PAGE101, correct):**
```markdown
> 问题仍然存在：创建上下文映射就这些了吗？可能吧。

高层视图提供了关于项目整体的相当多的知识...
```

The `![[FIG-3-7]]` line is simply removed. PAGE101 has no figure in the original.

**Prevention:** Step 3.5 invariant: "Every `![[../FIG/` reference MUST correspond to a figure that ACTUALLY APPEARS on the original book page."

---

## 3. Blockquote Boundary Drift

**Pages:** PAGE97, PAGE103 | **Pitfall:** #16

**Error Type A — Plain text wrongly wrapped in `>`:**

PAGE97: First paragraph + FIG3-4 + caption are plain body text in the original, but were wrapped in `>` in MDBOOK. Only the final paragraph (lighter background in original) should be blockquote.

**Error Type B — Callout text NOT wrapped in `>`:**

PAGE103: Whiteboard Time callout has two parts in the original (main text + "What if you find translations overly complex..." sidebar with left vertical bar). MDBOOK only blockquoted the first line, leaving the sidebar text un-blockquoted.

**Detection:** Render original PDF page, check for callout visual markers (gray background, left vertical bar, horizontal rules, different typeface).

**Fix — BEFORE (PAGE103, wrong):**
```markdown
> **白板时间**
> 为你项目限界上下文中某个有趣的集成方面创建一个翻译映射。

如果你发现翻译过于复杂，需要大量的数据复制和同步...
```

**Fix — AFTER (PAGE103, correct):**
```markdown
> **白板时间**
> 
> 为你项目限界上下文中某个有趣的集成方面创建一个翻译映射。
> 
> 如果你发现翻译过于复杂，需要大量的数据复制和同步，使你翻译后的对象看起来很像另一个模型中的对象怎么办？...
```

**Prevention:** Step 3.5 verify blockquote boundaries against original page callout markers.

---

## 4. Composite Figure — Duplicate Code Extraction

**Page:** PAGE103 | **Pitfall:** #26

**Error:** FIG3-7 is a composite diagram that visually includes Moderator box + HTTP headers + XML body + mapping arrows + caption. MDBOOK duplicated the XML from the figure as a separate ` ```xml ` code block.

**Detection:** `vision_analyze("Is the XML text on this page standalone or part of Figure 3.7?")`

**Fix — BEFORE (wrong, had both):**
```markdown
![[../FIG/MDBOOK260503-IMPLDDD-FIG-3-7.png]]

***图3.7*** *一个逻辑翻译映射...*

```xml
<discussion xmlns:...>
  ...
</discussion>
```
```

**Fix — AFTER (correct, only figure):**
```markdown
![[../FIG/MDBOOK260503-IMPLDDD-FIG-3-7.png]]

***图3.7*** *一个逻辑翻译映射，展示了表现形式状态（此处为XML）如何映射到本地模型中的值对象。*
```

The `![[FIG-3-7]]` image already contains the XML — no separate code block needed.

**Prevention:** Before writing code blocks, verify with `vision_analyze` whether the text is standalone or part of a composite figure.

---

## 5. Cross-Chapter Reference Format

**Page:** PAGE100 (and 13 others across Ch3) | **Pitfall:** #25

**Error:** Cross-chapter references written as bare text like `领域事件（8）` or `***限界上下文***（2）` (stars on name, number outside).

**Correct format:** ` ***领域事件（8）*** ` — three asterisks wrapping name+number TOGETHER, space before and after.

**Fix — BEFORE (wrong):**
```markdown
发布语言也用于 ***事件驱动架构（4）*** ，其中 ***领域事件（8）*** 作为消息传递给订阅的兴趣方。
```
(The spaces before/after `***` were missing.)

**Fix — AFTER (correct):**
```markdown
发布语言也用于 ***事件驱动架构（4）*** ，其中 ***领域事件（8）*** 作为消息传递给订阅的兴趣方。
```
(Three asterisks wrapping name+number together, one space before and after each group.)

**Detection:** Regex scan for `（\d+）` patterns in body text. Verify chapter number mapping against original book TOC.

**Prevention:** Content formatting rules table entry: ` ***名称（N）*** `.

---

## 6. Content Moved Across Pages

**Pages:** PAGE107→108 | **Pitfall:** #33

**Error:** PAGE107 has a heading `### ***为什么Discussion在两个上下文中都使用？***` near the bottom. The paragraph that follows this heading ("这是一个有趣的情况，因为概念名称 Discussion 在两个限界上下文中相同...") was moved to PAGE108. PAGE108 also had FIG3-9 caption text missing. Additionally, text from PAGE107's ending had been incorrectly placed on PAGE108.

**Detection:** Check every page-end heading. If followed by a body paragraph that starts on the NEXT page, the paragraph belongs on the heading's page. Also check cross-page flow: PAGE N's last line should naturally lead into PAGE N+1's first.

**Fix:** 
1. Move the paragraph "这是一个有趣的情况..." back to PAGE107, immediately after its heading
2. Restore PAGE108's missing FIG3-9 content
3. Fix the PAGE107→108 transition for natural sentence flow

**Prevention:** After any repair or bulk translation, verify PAGE N's last paragraph flows into PAGE N+1's first.

---

## 7. FIG Cropping — Full Page as Figure

**Chapter:** Ch4 (11 figures) + Ch3 (FIG3-5 initially) | **Pitfall:** #49

**Error:** Bulk extraction script called `page.get_pixmap()` and saved the entire page as the FIG file. Result: 1156×903px full-page renders with running headers, body text, page numbers surrounding the embedded figure.

**Detection (one-line check):**
```python
from PIL import Image
img = Image.open("FIG/...-FIG-4-1.png")
if img.size[1] > 800:
    print("⚠ Full page — needs 8-step cropping pipeline")
```

**Fix — 8-step pipeline:**
```
1. Locate figure page via text search ("Figure 3.5")
2. Render at 300 DPI, use vision to find bounding box as proportions
3. Refine boundaries with text block analysis
4. Set crop: fitz.Rect(w*0.08, h*0.60, w*0.92, h*0.97)
5. PIL ink detection + ImageOps.expand(border=50) for equal margins
6. Exclude horizontal separator rules
7. Visual compensation for asymmetric content
8. Vision verify: figure complete, no stray text, balanced margins
```

**Prevention:** After ANY batch figure extraction, run dimension check. Any FIG with h > 800 → route through 8-step pipeline.

---

## 8. FIG Cropping Quality — Caption Cut Off

**Page:** PAGE105 (FIG3-8) | **Pitfall:** #46

**Error:** FIG3-8 caption was cut off at the bottom — the last line of the caption text was missing.

**Detection:** `vision_analyze("Is the figure caption complete? Any text cut off at the bottom?")`

**Fix:** Re-crop with more generous bottom margin, verify caption completeness visually.

**Prevention:** Step 8 of the cropping pipeline — vision verify every extracted figure, checking caption completeness.

---

## 9. FIG Cropping — Top Edge Cropped

**Page:** PAGE106 (FIG3-9) | **Pitfall:** #46

**Error:** FIG3-9 had the top portion of its diagram cropped off — missing the upper participants/labels.

**Detection:** `vision_analyze("Is any part of the figure or its participants cut off at any edge?")`

**Fix:** Re-crop with expanded top boundary, run ink detection + equal margins, vision verify.

**Prevention:** Vision verification step must check ALL four edges for content loss.

---

## Error Frequency Summary (Ch3)

| Error Type | Pitfall | Pages Affected | Count |
|---|---|---|---|
| Layout reversal (figure not at correct position) | #14 | PAGE99, PAGE103, PAGE105, PAGE106, PAGE110 | 5 |
| Extra figure (not on original page) | #17 | PAGE99, PAGE101, PAGE111 | 3 |
| Blockquote boundary drift | #16 | PAGE97, PAGE103 | 2 |
| Composite figure code duplication | #26 | PAGE103 | 1 |
| Cross-chapter reference format | #25 | 13 pages across Ch3 | 13 |
| Content moved across pages | #33 | PAGE107, PAGE108 | 2 |
| FIG cropping: full page or incomplete | #46/#49 | FIG3-5, FIG3-8, FIG3-9, all 11 FIG4-* | 14+ |

**Key insight:** Nearly ALL errors would have been prevented by strict compliance with Step 3.5 (per-page visual verification) and the 8-step figure cropping pipeline. #18 (Step 3.5 non-compliance is root cause) is the most consequential pitfall.

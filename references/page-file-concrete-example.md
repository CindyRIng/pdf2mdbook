# PAGE File Concrete Examples

## Correct PAGE file structure

```markdown
# 链接

- **目录：** [[MDBOOK260503-IMPLDDD]]
- **上一页：** [[MDBOOK260503-IMPLDDD-PAGE1]]
- **下一页：** [[MDBOOK260503-IMPLDDD-PAGE3]]

# 内容

## 我能做DDD吗？

面对DDD的挑战，你需要知道如何成功。看看一个正在学习实践DDD的团队吧。

你应该对DDD有什么期待？... [完整翻译，不省略任何段落]
```

## Correct SECTION → PAGE relationship

**Leaf SECTION (section with no sub-sections):**
```markdown
# 页面

- [[MDBOOK260503-IMPLDDD-PAGE2]]
- [[MDBOOK260503-IMPLDDD-PAGE3]]
- [[MDBOOK260503-IMPLDDD-PAGE4]]
- [[MDBOOK260503-IMPLDDD-PAGE5]]
```

**Non-leaf SECTION (section with sub-sections):**
```markdown
# 子章节

- [[MDBOOK260503-IMPLDDD-SECTION1.4.1]]
- [[MDBOOK260503-IMPLDDD-SECTION1.4.2]]

# 页面

<待补充>
```

## Correct TOC entry format

```markdown
- ***1.1-我能做DDD吗？*** - [[MDBOOK260503-IMPLDDD-SECTION1.1]]-[[MDBOOK260503-IMPLDDD-PAGE2]]
```

Every entry has BOTH `-[[SECTION]]` AND `-[[PAGE起始页]]`.

## Image embedding

```markdown
![[../FIG/MDBOOK260503-IMPLDDD-FIG-0.5-2.png]]

**图 G.1** 展示限界上下文和相关统一语言的示意图。
```

Image goes BEFORE the caption, and both are part of the `# 内容` section.

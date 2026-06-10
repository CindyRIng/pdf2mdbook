# pdf2mdbook

将 PDF 书籍转换为结构化 Obsidian Markdown 三层体系（页→章→书），保留图片、格式与层级导航。

Convert PDF books into structured Obsidian Markdown with a three-tier system (Page → Chapter → Book), preserving images, formatting, and hierarchical navigation.

## 项目结构

```
pdf2mdbook/
├── SKILL.md              # 完整工作流文档（模板、4步转换流程、40+条 pitfalls）
├── references/           # 参考文档
│   ├── page-file-concrete-example.md
│   ├── pdf-page-mapping.md
│   ├── batch-rendering.md
│   ├── visual-translation-workflow.md
│   ├── parallel-translation-pattern.md
│   ├── parallel-vision-figure-scan.md
│   ├── section-filling.md
│   ├── content-audit.md
│   ├── image-extraction-failures.md
│   ├── equal-margin-figure-crop.md
│   └── search-files-roman-numeral.md
├── README.md
└── LICENSE               # MIT
```

## 使用方式

本项目是 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的一个 Skill，也可以作为独立参考文档使用。

### 作为 Hermes Skill

加载 skill 后即可使用 4 步工作流：

1. **初始化** — 提取书籍信息，创建目录骨架
2. **构建 TOC** — 解析 PDF 目录，生成完整章节层级与 PAGE 骨架
3. **提取页面内容** — 逐页提取文本（pymupdf）并翻译为中文，嵌入图片
4. **自底向上填充** — 从叶子节点向上填写摘要、页面链接、子章节链接

### 作为独立参考

直接阅读 `SKILL.md` 了解完整的转换方法论、模板格式、图片提取流程和常见坑。

## 许可证

MIT License — 详见 [LICENSE](LICENSE)

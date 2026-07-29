# Raw Sources（原始文档）

> 这个目录存放所有原始文档（PDF、文章、网页剪藏、笔记等）。
> **只读不修改** —— 这是你的知识来源的信任锚点。
> LLM 会读取这里的文档，提取知识，然后反映到 wiki 层。

## 使用方式

1. 将新文档放入此目录
2. 告知 LLM："帮我处理 `raw/sources/xxx`"
3. LLM 会完成：读取 → 摘要 → 创建/更新 wiki 页面 → 更新 index 和 log

## 支持格式

- `.md`（Markdown）
- `.pdf`（需要 OCR 或文本提取工具）
- `.txt`
- 网页剪藏（Obsidian Web Clipper 导出的 markdown）

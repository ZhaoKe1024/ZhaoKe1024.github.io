---
title: "Hello, World：这个博客写什么"
date: 2026-11-13 20:00:00 +0800
permalink: /posts/2026/11/hello-world/
tags:
  - blog
  - llm4opt
  - ai-for-education
---

这是第一篇文章，也是一份**写作模板**：把本文件复制改名就能写新博客。

我打算在这里记录三类内容：

1. **LLM4Opt 笔记** —— 优化问题 benchmark、Rubrics 评测、求解器接入的踩坑记录。
2. **工具与工程** —— PyTorch 训练框架、音频/异常检测流程、DSH 插件开发。
3. **科研杂记** —— 读博期间的读书笔记与方法论。

## 常用写法示例

代码块（先写三反引号，再写语言名）：

```python
from optskills import solve

result = solve("一家工厂如何安排生产使利润最大？")
print(result.status, result.objective)
```

行内代码用反引号，比如 `_config.yml`。

数学公式（模板已内置 MathJax）：

$$
\max_{x} \; c^\top x \quad \text{s.t.} \; Ax \le b,\; x \ge 0
$$

图片（把图放到 `images/` 下，然后）：

```markdown
![描述文字](/images/your-figure.png)
```

表格：

| 方法 | 求解器 | 状态 |
|:-----|:-------|:-----|
| LP   | HiGHS  | ✅   |
| MIP  | COPT   | ✅   |

## 新建文章的规则

- 文件放在 `_posts/`，文件名必须是 `YYYY-MM-DD-标题.md`（日期即发布时间）。
- 开头 `---` 之间是 YAML：`title`、`date`、`tags` 必填；`permalink` 可选。
- 草稿放在 `_drafts/`（文件名无需日期，`jekyll serve --drafts` 预览）。
- 文章会自动出现在 [/year-archive/](/year-archive/) 和 RSS 里。

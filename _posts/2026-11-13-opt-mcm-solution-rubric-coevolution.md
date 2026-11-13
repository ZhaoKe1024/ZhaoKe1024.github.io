---
title: "OPT-MCM 精读：一个 Task 是怎么定义的，Rubric 又是怎么被“共同演化”出来的"
date: 2026-11-13 22:10:00 +0800
permalink: /posts/2026/11/opt-mcm-solution-rubric-coevolution/
tags:
  - opt-mcm
  - llm4opt
  - benchmark
  - rubric
  - agent
categories:
  - paper-reading
excerpt: '开放式优化任务没有唯一解，目标值评不了。OPT-MCM 的答案是：把 Task 定义成四元组，让解法和评分标准一起演化。这篇拆解 Solving 与 Rubric Generation 两步的输入输出，以及每一轮 rubric 到底靠什么被 refine。'
---

> 论文：**OPT-MCM: A Near Real-World Benchmark for End-to-End Optimization Agents with Co-Evolving Rubrics**（KDD 2027）
> 代码：<https://github.com/ZhaoKe1024/OPT-MCM>

## 0. 为什么需要一个新的 benchmark

MIPLIB、Netlib、OR-Library 这类经典基准评的是**算法**：模型给定，只剩求解。
NL4Opt、MIPLIB-NL、OptiVerse、OPT-Engine 评的是**建模**：自然语言 → 可执行的数学规划，正确性靠目标值对得上。

但真实的优化项目是这样开始的：

> “我们想在华东几个城市做一条两周的研学线路，预算每人 8000，尽量多看博物馆，但要避开周一闭馆，而且要照顾两位 60 岁以上老人……”

题干松、信息不全、约束隐含、目标还会变，而且**多个建模路线都能算对**。这类任务没法用“目标值对不对”来判分——你甚至不知道目标函数该是什么。于是 OPT-MCM 提出 **near real-world optimization task（近真实优化任务）**：目标明确、材料齐备，但模型与解法完全开放。

## 1. Task 是怎么定义的

一条 task 不是一段题面，而是一个四元组：

$$
\text{Task}_p = \bigl(p,\; A_p,\; D_p,\; R_p\bigr)
$$

| 符号 | 含义 | 谁来产生 |
|:--|:--|:--|
| \(p\) | 原始任务陈述（竞赛题面原文） | 数据源 |
| \(A_p\) | **结构化任务分析文档**（shared semantic anchor） | LLM 生成 + 人工校验 |
| \(D_p\) | 附件数据（表格、JSON 图、CSV 等） | 数据源 |
| \(R_p\) | 任务级、**solution-route-invariant** 的 rubric | Solution–Rubric Co-Evolution |

关键在 \(A_p\)。题面往往长达十几页、夹着附件和“背景介绍”，如果让不同阶段的模型各自去读，理解会发散。所以先把题面重组成一份**结构化文档**，它只做重组、不引入任何题面之外的信息：

- 子任务分解（题目要求的 (1)(2)(3) 分别要交付什么）
- 已知条件 / 附件清单
- 任务目标（要优化什么、要回答什么）
- 关键概念与领域术语
- **显式约束与隐式约束**（这是 rubric 的主要来源）
- 子任务之间的依赖关系
- 期望输出形式（表格？决策建议？敏感性分析？）

同一份 \(A_p\) 既喂给评分构建流程，也喂给被评测的 agent。这样“出题人”和“答题人”读到的是同一个任务理解，避免理解偏差被当成能力差异。

数据源选了 **CUMCM（全国大学生数学建模竞赛）+ CPMCM（研究生数学建模竞赛）**，最终 **34 个任务**。不用 MCM/ICM 的原因很实在：MCM 很多题要选手自己去采数据，而“能不能搜到数据”会盖过“会不会建模”这个被测能力。

## 2. 评分的对象是“交付物”，不是“一个数”

agent 要跑完整生命周期：读题 → 假设 → 建模 → 选算法 → 求解 → 验证 → 给出决策建议。
交付物是一份**结构化文本文档**（Markdown/LaTeX，可含表格与公式），不要求图片或多媒体。

评分由 LLM-as-a-Judge 完成：

$$
\mathrm{Score}(x_{p,k}) = \mathcal{E}\bigl(p,\; A_p,\; R_p,\; x_{p,k}\bigr) \in [0, 100]
$$

其中 \(x_{p,k}\) 是 agent 在路线 \(b_{p,k}\) 下产出的交付物。

## 3. Solving：Agent 输入什么、交付什么

### 3.1 输入

- **问题目录**：题面 + \(D_p\) 附件 + `submission_schema.md`（交付物格式约定）
- **共享分析** \(A_p\)
- **方向说明**（direction）：本分支的建模路线、方法族、必须给出的证据、主要风险

默认 solver 是 **Claude Code + 我们写的 optimization skills**。注意：agent 拿到的是“目录 + 分析”，不是“一段 prompt 里的数值”，它得自己去读附件、自己整理中间数据。

### 3.2 交付

在指定输出目录下产出：

```text
<direction_output_dir>/
├── code/                       # 可复现代码
├── results/                    # 所有重要计算结果（不只是 print）
│   └── solution_report.md      # 主报告（评分对象）
└── logs/                       # 运行日志
```

`solution_report.md` 就是被评测的 \(x_{p,k}\)。约束里明确要求：**不要写“论文式”的泛泛总结，要可运行、可核对的结果**；某个子问题实在解不出来，也要给出最站得住脚的交付并写明局限。

### 3.3 先探索路线，再出解

直接让模型“做这道题”，它只会给你一条最顺手的路线。而 rubric 想要 route-invariant，就必须先看到多条**实质不同**的路线。所以求解前先跑一个 **solution direction generation**：

输入：问题目录、\(A_p\)；输出：4–6 条方向，每条带

```json
{
  "direction_id": "direction_02_network_flow",
  "label": "时间展开网络流",
  "method_family": "network flow / MIP",
  "rationale": "...",
  "best_for": ["多城市路径", "时间窗"],
  "core_commitments": ["离散化时间", "流量守恒"],
  "solver_focus": ["Gurobi", "OR-Tools"],
  "required_evidence": ["流量守恒检验", "界与 gap"],
  "key_risks": ["时间离散粒度导致不可行"]
}
```

规则写得很硬：方向之间不能只差遣词或方法名；不能提前给数值答案或完整代码。然后每条方向各起一个 branch solver，并行跑出 \(K\) 份交付物。

**这 \(K\) 份解不是参考答案，而是“证据”。** 它们暴露了任务的隐藏约束、常见失败模式、哪些建模决策真正区分好坏——这些正是 rubric 需要的素材。

## 4. Rubric Generation：一轮到底输入什么、交付什么

### 4.1 初始化（每条分支一份粗 rubric）

输入：方向描述 + \(A_p\) + branch id / family
输出：一份**刻意保持粗粒度**的 rubric JSON

```json
{
  "dimensions": [
    {
      "dimension_id": "D1",
      "name": "模型与假设",
      "weight": 0.25,
      "criteria": [
        {
          "criterion_id": "D1_C1",
          "name": "隐式约束是否被显式化",
          "description": "...",
          "full_score": 10,
          "required_evidence": ["..."],
          "acceptable_variants": ["..."],
          "failure_modes": ["把周一闭馆当成软约束"],
          "scoring_guidance": "...",
          "truthfulness_notes": ["不得臆造未出现的证据"]
        }
      ]
    }
  ],
  "hard_constraints": ["..."],
  "provenance": {"source": "branch_prompt_bootstrap", "version": 1}
}
```

约束：4–6 个维度、每维 1–3 条 criterion、权重非负且和为 1、不用 0/1/2/3 那种粗档描述、不把满分绑定到某个具体算法名。

### 4.2 每一轮怎么 refine

同一轮里，评估与精修是**一次调用、两轮返回**：上下文固定为

```text
(problem_id, branch_id, branch_family, A_p, R^{t-1}, x)
```

第一轮返回：

```json
{
  "evaluation_md": "中文评语：总分 + 逐条给分理由",
  "feedback_md":   "给 solver 的修改指令"
}
```

第二轮（同样的上下文、同样的判断）返回精修后的 rubric：

```json
{
  "refined_rubric": {
    "criteria": [
      {
        "criterion_id": "C3",
        "title": "参数取值是否有数据支撑",
        "criterion_intent": "...",
        "why_this_is_branch_general": "换了同族解法的另一个人也成立",
        "full_score": 8,
        "evaluation_method": ["核对引用的附件列", "检查是否复算"],
        "key_checks": ["..."],
        "failure_tests": ["若系数无出处 → 不给分"],
        "confidence_scope": "只判它声称的部分",
        "scoring_guidance": "..."
      }
    ]
  }
}
```

**精修的两条操作**（这是全文最实操的部分）：

1. **粗维度 → 窄检查**：把“建模质量 20 分”拆成“假设是否前后一致传播”“参数是否有数据出处”“改进在给定约束下是否仍可行”这类可直接核对的小项。
2. **每条 criterion 补 `key_checks` 和 `failure_tests`**：前者说“怎么查”，后者说“什么情况直接扣分”。这让评分从“印象分”变成可复现的判断。

同时有三条硬约束防止 rubric 漂移：

- 判**正确性与逻辑有效性**，不判文笔；
- 不奖励“有这一节/有这张表/写了这个方法名”；
- **分支通用**：不许出现答案特定的常量、文件名、迭代次数、章节命名。

于是每轮其实是一个闭环：

```text
R^{t-1} 评分 x_t  ──►  feedback_md  ──►  solver 改解 ──► x_{t+1}
      │
      └──► refined_rubric R^{t} ──► 下一轮用它评分
```

**解变好会逼出更细的 rubric，更细的 rubric 又会给出更有信息量的 feedback 去逼解变好。** 这就是 Solution–Rubric Co-Evolution 的字面意思。跑满 \(T\) 轮后得到 \(R^{(T)}_{p,k}\)。

### 4.3 从 K 份分支 rubric 聚合成一份任务级 rubric

聚合不是“拼盘”，而是**抽象**。做法是让一个 planner 只做**结构规划**（不许改写措辞、不许改语义、不许重复计分），把保留的 criterion 组织成两类 family：

- **`or_group`：同一种能力的多种合法实现**。比如“硬约束编码”在精确优化分支是 MIP 变量，在网络流分支是可行状态/弧，在启发式分支是解的编码。同一个能力，三条路线各给一种实现——**只按提交实际走的那条路线评分**，不要求同时满足三条。
- **`sum`：跨路线共享的义务**。比如“输出核验与可复现”，三条路线都会写，那就只保留一条，其余通过显式 coverage 映射挂过去。

评分时先做**源内归一化**（避免某条分支的满分质量级不同导致总分被拉扯）：

$$
\tilde{s}(c) = 100\cdot\frac{s(c)}{M_{k(c)}},\qquad
\tilde{m}(c) = 100\cdot\frac{m(c)}{M_{k(c)}}
$$

`sum` family：
$$
S(F)=\sum_{c\in F}\tilde{s}(c),\qquad M(F)=\sum_{c\in F}\tilde{m}(c)
$$

`or_group` family（取支撑最好的那条实现）：
$$
S(F)=M(F)\cdot\max_{r\in\mathcal{R}(F)}
\frac{\sum_{c\in r}\tilde{s}(c)}{\sum_{c\in r}\tilde{m}(c)}
$$

总分：
$$
\mathrm{Score}(x)=100\cdot
\frac{\sum_{F\in\mathcal{F}_p}S(F)}{\sum_{F\in\mathcal{F}_p}M(F)}
$$

这套归一化 + family 结构解决了一个很实际的公平性问题：**不能因为一个 agent 没用 MIP，就在“MIP 建模”那条上扣它分。**

## 5. 实验里最值得看的两组数

- **专家一致性**：自动构建的 rubric 与专家打分 **Spearman = 0.82**。
- **可靠性**：用“逐级退化”的获奖报告构造可控质量验证集（原始 + 四档退化 = 5 档），rubric 能在异构任务上稳定区分质量档位，说明它不是因为“字多”就给高分。

agent 侧的结论也很直白：**设计良好的 workflow + 自主 agent 组合在建模、执行、验证三维度同时最强**；而 agent 在真实任务上的瓶颈依然是“建立稳健的模型”和“把结果算对”，不是写报告。

## 6. 我的几点体会

1. **把“分析文档”作为锚点**，比反复优化 prompt 有效得多。它一次性消除了“同一道题被不同模型理解成不同题”的噪声。
2. **rubric 的第一版必须粗**。一上来就写细则，会把自己对题目的先验偏见固化；先粗、再用真实解去“撞”出细项，才是数据驱动的。
3. **route-invariance 是这套方法的命门**。它靠两件事保证：分支内精修时禁止答案特定描述；分支间用 or_group 而不是加总。
4. **代价**：一个任务要跑 K 条路线 × T 轮，LLM 调用量不小；rubric 有可能向“更常见、更好写的路线”收敛，需要人工审核兜底（我们确实做了专家复核）。

---

如果你在做 agent 评测，这套“Task = 题面 + 分析 + 附件 + rubric”的四元组定义，其实可以直接搬到别的开放式任务上（科研写作、数据科学、方案设计）——只要那个任务的产出是可读的交付物。

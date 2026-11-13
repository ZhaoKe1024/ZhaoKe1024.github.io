---
title: "OptSkills：按 archetype 聚类，再把轨迹蒸馏成可复用 Skill"
date: 2026-11-13 22:20:00 +0800
permalink: /posts/2026/11/optskills-archetype-cluster-distillation/
tags:
  - optskills
  - llm4opt
  - skill-learning
  - clustering
  - distillation
categories:
  - paper-reading
excerpt: '同一类优化问题换个场景描述就换个说法，按文本相似度复用经验必然失灵。OptSkills 的答案是：先把问题投影到 archetype 表示空间聚类，再在类簇上做多求解器 rollout、蒸馏 skill，并让 skill 库在新问题上自我精修与扩张。'
---

> 论文：**OptSkills: Learning Generalizable Optimization Skills from Problem Archetypes via Cluster-Based Distillation**（EMNLP 2026）
> 代码：<https://github.com/fujiwaranoM0kou/OptSkills>

## 0. 问题：为什么“案例级复用”不够

LLM 做优化建模的三种主流做法——后训练（ORLM/OptMATH/LLMOPT/SIRL）、多智能体工作流（CoE/OptiMUS/ORMind/ORThought）、经验增强（AlphaOPT、LEAN-LLM-OPT）——都在解决“怎么把知识搬进推理过程”。但经验增强类方法大多是**案例级**的：检索一个相似题目，照抄它的建模与代码。

案例级复用有三个先天问题：

- **对叙述变化敏感**：“排班”“值班”“轮岗”是同一类问题，文本却几乎不重叠；
- **蒸馏不出工作流知识**：只学到“这道题这么写”，学不到“这类题的建模动作序列”；
- **难以应对分布漂移**：新领域里出现同类问题的语义变体，或干脆是新问题类型时，旧案例失效。

所以本文换了一个视角：**按问题的规范结构（archetype）而不是表面文本相似度来组织经验。**

先给一个经验证据：把 Nano-CO 上的问题分别用「原始题面 embedding」和「archetype embedding」投影，后者同类样本更紧凑、类间更分离；最近邻检索的 Hit@1 与 MAP@5 也明显更高。也就是说，**archetype 表示更适合做检索与聚类的基础。**

## 1. archetype 是什么

一个优化问题 archetype \(a\in\mathcal{A}\)，是从具体应用场景里抽象出来的**规范优化结构**。给定 archetype \(a\)，它的自然语言实例记作

$$
x = g_u(a)
$$

其中 \(u\) 是具体场景，\(g_u\) 是“把这个结构用某个场景说出来”的 verbalization 过程。理想情况下，x 和它的 archetype 应当诱导出同一个优化模型：

$$
f_{\mathrm{model}}(a) = f_{\mathrm{model}}\bigl(g_u(a)\bigr)
$$

翻译成人话：**建模应该对场景叙述不变**。同类问题的决策变量、目标、约束模式、参数配置是同构的，只是故事不同。

### 1.1 怎么得到 archetype 表示

用一个 LLM extractor 从题目 \(x_i\) 抽两样东西：

- **ingredients \(\kappa_i\)**：抽象层面的变量、约束、目标（刻意忽略实体名、场景背景）；
- **edited problem \(\tilde{x}_i\)**：把场景叙述与实体名替换成通用占位符，但**保留**原始参数、约束与数值。

两者分别取 embedding \(w_i, v_i\)，加权归一化：

$$
e_i = \mathrm{Norm}\bigl(\alpha\cdot w_i + (1-\alpha)\cdot v_i\bigr)
$$

embedding 模型是 Qwen3-Embedding-v3，\(\alpha\in[0,1]\) 控制“结构”与“改写后文本”的相对贡献。

### 1.2 怎么聚类

对归一化后的 \(e_i\) 直接上 **DBSCAN**（余弦距离）：

$$
\mathcal{C} = \mathrm{DBSCAN}\bigl(\{e_i\}_{i=1}^{N};\ \epsilon,\ m\bigr)
$$

选 DBSCAN 而不是 k-means，是因为它不需要预先知道簇数、能标出噪声点——训练集里本来就有一些“孤例”不该被硬塞进某个 archetype。每个簇 \(C\) 即一组同 archetype 的问题。

## 2. 类簇上怎么演化：Phase I 的四步

整体是“先建库、再学、最后用”三段。Phase I 建初始 skill 库 \(\mathcal{M}_0\)，有四个动作：

### (1) Solver Portfolio Rollout —— 同一道题，多路线跑

不同建模范式与求解器配置在同一道题上的适用性、稳定性差别很大。所以先让一个 LLM-based solver selector 从候选池 \(\mathcal{B}\) 里选出 top-k 个求解器 \(\mathcal{B}_i\)，然后：

1. 为每个 \(b\in\mathcal{B}_i\) 生成优化模型 \(z_i^b\)；
2. 迭代生成可执行代码 \(c_{i,t}^b\) 并接收执行观测 \(o_{i,t}^b\)；
3. 经过 \(T_i^b\) 轮后得到可行解 \(\hat{y}_i^b\)。

一条轨迹记作

$$
\tau_i^b = \bigl(z_i^b,\ c_{i,1}^b, o_{i,1}^b,\ c_{i,2}^b, o_{i,2}^b,\ \dots,\ c_{i,T_i^b}^b,\ \hat{y}_i^b\bigr)
$$

轨迹集合 \(\mathcal{T}_i=\{\tau_i^b\}\)，按答案是否等于 \(y_i^\ast\) 分成正集 \(\mathcal{T}_i^+\) 与负集 \(\mathcal{T}_i^-\)。

> 这里是全流程最贵的一步，但也是信息量最大的：**正轨迹告诉你“什么配置能跑通”，负轨迹告诉你“什么会崩”**。

### (2) Trajectory Analysis —— 把轨迹压成 SOP + 坑

一个 LLM skill analyst 读正负轨迹，输出

$$
q_i = A_\eta(\mathcal{T}_i^+,\mathcal{T}_i^-) = (r_i,\ p_i)
$$

- \(r_i\)：从正轨迹里抽出的**标准作业流程（SOP）**——变量定义、目标构造、约束写法、求解器配置、状态检查、结果解析；
- \(p_i\)：从负轨迹里总结的**常见坑**。

\(q_i\) 是后续蒸馏的原子单位。

### (3) Cluster-Based Skill Distillation —— 在类簇上蒸馏

对每个簇 \(C\)，skill builder 聚合簇内所有 \(q_i\)：

$$
s_C = B_\eta\bigl(\{q_i \mid i\in C\}\bigr),\qquad
\mathcal{M}_0 = \{s_C \mid C\in\mathcal{C}\}
$$

这样得到的 skill 是**跨样本**的：同一 archetype 下不同实例的正负经验被合并，instance-specific 的细节被压掉，留下可复用的过程性知识。

一个 skill 由三元组定义：

$$
s=(m,\ w,\ p)
$$

- \(m\)：metadata（名字、一句话适用说明）——用于检索；
- \(w\)：建模求解工作流；
- \(p\)：常见坑。

### (4) 得到的规模

150 个样本做聚类蒸馏，得到 **46 个 skill**（初始库）。对照组不用聚类、逐样本抽 skill，得到 **129 个**——但其中 **81 个（62.8%）在 5 个 benchmark 上从未被调用**。而 46 个聚类 skill 里只有 **7 个（15.2%）** 从未被调用。

这个对比很说明问题：**skill 库不是越大越好，语义重复会让检索选错。**

## 3. Phase II：新问题上怎么 refine / expand

拿到新问题 \(x\)，先做同样的抽取与改写，然后由 skill selector 比对库中所有 skill 的**名字与描述**，选出候选；再交给一个 judge 判定“这个 skill 能不能用来解这道题”。

### 3.1 接受 → Skill Refinement

若接受，用被选中的 \(s^\star=(m^\star,w^\star,p^\star)\) 作为过程性指导去解 \(x\)，得到新轨迹 \(\tau\)：

- **正轨迹** → 强化 \(w^\star\) 里可复用的 SOP；
- **负轨迹** → 往 \(p^\star\) 里补失败证据。

轨迹先经 \(A_\eta\) 压成分析 \(q\)，再由 skill refiner 产出 \(s_{\text{refined}}\)。refiner **保留 metadata 与适用边界**，只更新 workflow 与 pitfalls，然后原地替换库中的 \(s^\star\)。

### 3.2 不接受 → Skill Expansion

若没有合适 skill，就把它当作“新 archetype 的候选”，跑一次 solver portfolio rollout 得到 \(\mathcal{T}_x\)，同样压成 \(q\)。

关键护栏：**只有至少存在一条正轨迹时，才允许扩张**（\(\mathcal{M}_{t+1}=\mathcal{M}_t\cup\{s_{\text{new}}\}\)）。否则库会被“来自失败尝试的伪知识”污染。

### 3.3 测试期

测试时库 \(\mathcal{M}^\ast\) 冻结，不再更新：抽取 \(\kappa,\tilde{x}\) → selector 按名字/描述检索 skill → 注入 agent 作为过程指导 → 生成解 → 对答案。

## 4. 实验：几个值得看的曲线与数字

### 4.1 主结果（5 个 benchmark，Pass@1）

| 方法 | Micro-Avg. | OptiBench | Mamo.C | OptMATH | IndustryOR | ComplexOR |
|:--|--:|--:|--:|--:|--:|--:|
| DeepSeek-V3.2（裸模型） | 49.45 | 62.64 | 27.49 | 40.36 | 30.00 | 55.56 |
| GPT-5.4 | 57.82 | 69.09 | 47.39 | 46.39 | 29.00 | 66.67 |
| AlphaOPT（经验增强） | 58.55 | 70.25 | 54.50 | 36.75 | 32.00 | 61.11 |
| Trace2Skill（强 skill 基线） | 63.46 | 75.21 | 54.03 | 49.40 | 34.00 | 72.22 |
| **OptSkills-D** | **68.27** | **77.02** | **63.51** | **61.45** | **36.00** | **72.22** |
| OptSkills-Q（换 Qwen3-235B） | 61.46 | 71.74 | 53.55 | 54.22 | 27.00 | 66.67 |

OptSkills-D 比最强基线 Trace2Skill 高 **+4.81**。换 backbone 到 Qwen3 仍能到 61.46，说明增益来自系统本身而不是底座模型。

### 4.2 skill 数量 vs benchmark 性能

在 OptiBench 上，学习过程中设了 4 个 checkpoint，对应 skill 库规模 46 → 49 → 53 → 56：

| skill 数 | 46 | 49 | 53 | 56 |
|:--|--:|--:|--:|--:|
| OptiBench SA (%) | 74.71 | 74.05 | **75.37** | **77.02** |

**整体是上升的，但在 49 处有一次回落。** 这说明“扩库”不是单调增益：新 skill 与旧 skill 的语义重叠会短暂干扰检索；只有覆盖度真正扩开后才转而涨。这也是为什么我们坚持“聚类蒸馏”而不是“每样本一个 skill”。

> 说明：论文里没有直接以“学习迭代轮次”为横轴的曲线，而是用 skill 库规模的 checkpoint 表示学习进程；这条非单调轨迹就是迭代-性能关系的代理指标。

### 4.3 消融：每个模块的贡献

| 配置 | OptiBench | Mamo.C | OptMATH | IndustryOR | ComplexOR |
|:--|--:|--:|--:|--:|--:|
| 仅 Tool-Call Agent | 73.22 | 53.08 | 54.21 | 34.00 | 66.67 |
| + 逐样本 skill 蒸馏 | 69.75 | 56.40 | 59.64 | 32.00 | 66.67 |
| + 聚类蒸馏 | 74.71 | 59.24 | 60.84 | 35.00 | 72.22 |
| + Skill Learning（ID） | **77.02** | **63.51** | **61.45** | **36.00** | **72.22** |

逐样本 skill 反而在 OptiBench 上掉点，聚类蒸馏把它救回来——再次印证“冗余 skill 会伤害检索”。

### 4.4 OOD：NLCO 上的迁移

用模板合成出 OOD 数据集 Nano-CO（问题类型分布与训练集不同）做 skill learning：

| 方法 | NLCO Accuracy (%) |
|:--|--:|
| Tool-Call Agent | 62.60 |
| OptSkills（学习前，46 skills） | 68.56 |
| OptSkills（学习后，46 + 47 = 93 skills） | **72.79** |

相比 baseline **+10.19**。而且成功轨迹里新 skill 的使用率显著高于失败轨迹（1535 个成功样本中 644 个用到新 skill；576 个失败样本中只有 176 个用到）。

### 4.5 MIPLIB-NL：大规模真实问题

MIPLIB-NL 是大规模工业实例（变量、约束都很多），是这类方法最容易翻车的地方：

- **OptSkills-D 求解准确率 26.91%**；
- 比 MIPLIB-NL 论文里报告的 **DeepSeek-V3.2-Thinking（22.38%）高 4.53 个点**；
- **在被正确求解的实例中，决策变量维度最高达到 87,482。**

这说明 archetype 级的过程知识在大规模问题上依然能迁移，而不是只在“教科书题”上有效。

## 5. 我的理解：这套设计真正解决了什么

1. **把“经验”的粒度从案例提升到工作流。** skill 的 \(w\) 是一串建模动作，不是一段答案文本，所以能扛叙述变化。
2. **聚类不是工程细节，而是性能来源。** 46 vs 129、15.2% vs 62.8% 的未使用率对比，直接解释了为什么“多”不等于“好”。
3. **refine 与 expand 的分工很关键。** 前者在旧 archetype 内做增量修正，后者只在有正证据时扩张——这避免了库被噪声污染。
4. **可迁移性靠“有正轨迹才扩库”这条硬规则兜底。** 这条规则看着不起眼，但它是 OOD 场景下不崩的原因。

---

后续我打算把这套 skill 格式（metadata / workflow / pitfalls）推广到更多求解器后端上，也欢迎直接拿 OptSkills 的 skill 库去试你自己的题。

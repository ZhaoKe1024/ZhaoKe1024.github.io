---
title: "CAMTP：先约束、后求解——IJCAI'25 行程规划竞赛的冠军方案"
date: 2026-11-13 22:40:00 +0800
permalink: /posts/2026/11/camtp-travel-planning-champion/
tags:
  - camtp
  - travel-planning
  - multi-agent
  - dsl
  - constraint
categories:
  - paper-reading
excerpt: '旅行规划是典型的“硬约束 + 软偏好”优化：约束抽取不可靠、LLM 又不会做空间推理。CAMTP 的做法是把自然语言先压成 44 类结构化约束，再代码化转成 DSL，最后用六个专职 agent + VRP-TW 求解器拼出行程。'
---

> 论文：**A Constraint-Aware Multi-Agent Optimization Framework with Robust Domain Specific Language Generation for Travel Planning**
> *Frontiers of Computer Science*，Just Accepted，DOI: [10.1007/s11704-026-52005-y](https://doi.org/10.1007/s11704-026-52005-y)
> 代码：<https://github.com/zhuyiyi-123/CAMTP>
> 说明：我是本文共同作者，这篇主要记录我对框架设计的理解（竞赛方案 + 工程取舍）。

## 0. 任务形式化：什么算一个“好行程”

用自然语言 \\(Q\\) 描述需求，要在城市集合 \\(C\\)、POI 集合 \\(P\\) 上构造多日行程

$$
\pi=\langle a_1,a_2,\dots,a_T\rangle,\qquad a_t\in P
$$

评价 \\(\pi\\) 好坏有三层：

- **环境可行性**：POI 是否存在、营业时间、交通是否可达（真实世界约束）；
- **逻辑约束**：\\(\pi\vDash\varphi_i\\)，对全部用户硬约束 \\(\varphi_i\in\mathcal{L}_{DSL}\\)（预算上限、到达截止时间……）；
- **偏好优化**：最大化/最小化软目标 \\(f_j(\pi)\\)（景点数、市内交通时间……）。

形式化地：

$$
\pi^\star=\arg\max_{\pi\in\mathcal{S}(Q)}\ \mathcal{U}(\pi)
\quad\text{s.t.}\quad \pi\ \text{可行},\ \pi\vDash\{\varphi_i\}
$$

其中 \\(\mathcal{U}\\) 是由软目标构成的复合效用，\\(\mathcal{S}(Q)\\) 是查询诱导的搜索空间。

这个形式化看起来平平无奇，但它把两类错误清楚地分开了：**约束没满足（硬错）** 与 **偏好没优化好（软亏）**。竞赛评分也是这么设计的：EPR / C-LPR / FPR 三个硬约束指标 + DAV / ATT / DDR 三个软偏好指标，加权得到总分。

## 1. 两个真问题

**问题一：从自然语言里抽约束不可靠。**
用户说“别太贵”“离景点近一点”“住得舒服点”时，约束是隐含的；地名还经常是残缺的（“西湖边上”）。ChinaTravel 的 NL2DSL 路线存在三个毛病：约束覆盖不全、强依赖 prompt、用户表达有歧义。

**问题二：LLM 没有空间感知。**
就算模型把约束都抽对了，它也不擅长算最短路径、判断多个 POI 的地理与时间可达性。直接把规划交给 LLM 生成，容易出现“看着合理但不可执行”的行程。

于是 CAMTP 有两条主线：

1. **NL → 约束 → DSL（NL2C2DSL）**：把“自然语言直接生成 DSL”这条难路，拆成“抽取约束”+“代码化转 DSL”两条简单路；
2. **多智能体 + 传统优化算法**：让 LLM 负责语义与偏好，让 OR 求解器负责组合与可行性。

## 2. NL2C2DSL：为什么中间要夹一个“约束层”

原来的三步 NL2DSL 是：自然语言 → 结构化约束 → DSL。我们的改造集中在中间那一步，并把它做实：

### (1) 重设计结构化约束，收敛到 44 类

从 ChinaTravel 数据集与两个竞赛 phase 的数据里归纳出 **44 个约束类别**，并给每一类固定字段。分 8 大组：

| 组 | 代表约束 |
|:--|:--|
| 行程格式 | `start city`、`target city`、`days`、`people number` |
| 预算 | `total cost`、`intercity cost`、`accommodation cost`、`restaurant cost`、`attraction cost`、`inner city cost` |
| 城际交通 | `go must type`、`back must not type`、`must inner city transportation`、`distance transport` … |
| 住宿 | `must / must not accommodation name`、`must room type`、`distance hotel` … |
| 餐饮 | `must restaurant type`、`must not restaurant name`、`restaurant stay time` … |
| 景点 | `must attraction name any`、`attraction between`、`attraction seq`、`attraction stay time` … |
| 时间 | `activate start time`、`activate end time` |
| 其他 | 城市间交通方式限制、人数与房间匹配等 |

这套分类的好处是：**每类都有“返回内容规范”和“输出格式样例”两个字段**，抽取时的稳定性明显提高；而且这类结构化约束**可以直接代码化转成 DSL**。

### (2) C2DSL 用代码映射，而不是再问一次 LLM

原方案在“约束 → DSL”这步又走一次 LLM 推理，既有 token 开销，又引入二次幻觉。我们改成**基于重设计约束的直接代码转换**：约束已经是机器可读结构，映射规则是确定性的。

这一步是整个 pipeline 里性价比最高的改动：**没有 LLM 参与，就没有了新的不确定性。**

### (3) 检索增强修正（RAR）

抽出来的地名可能不完整、不规范，会导致 DSL 校验失败。做法是把抽到的地名当查询，在候选地址语料里用 **BM25** 排序：

$$
\mathrm{score}(q,d)=\sum_{t\in q}\mathrm{IDF}(t)\cdot
\frac{f(t,d)\cdot(k_1+1)}{f(t,d)+k_1\cdot\bigl(1-b+b\cdot\frac{|d|}{\mathrm{avgdl}}\bigr)}
$$

取最高分地址回填修正约束。**无监督、不需要标注数据、对残缺查询鲁棒**——这比再训一个小模型做实体链接划算得多。

### (4) SFT 微调抽取器

用 ChinaTravel 的 easy/medium/human/human1000 + 两个 phase 的竞赛数据构造 NL2C 训练集：

- 有 DSL 标注的数据：把 DSL 标注**代码化反推**成结构化约束作为监督；
- 竞赛数据：**采样重组约束**生成合成训练数据（原始查询留作评测，避免数据泄漏）；
- human1000 无标注：用 DeepSeek-V3 在 NL2C2DSL 管线下补标注；

训练用 LoRA（rank 64 / alpha 128，全线性层，3 epoch，cosine 衰减，lr \\(2\times10^{-4}\\)，warmup 0.03）。

## 3. 多智能体：谁负责什么

框架分三块：环境模块（NL2C2DSL + SFT）、思考模块（约束分解与任务分派）、动作模块（六个专职 agent）。

### 3.1 思考模块

拿到结构化约束后，把任务拆到五个方面：**城际交通选择、景点推荐、路径规划、餐饮分配、住宿安排**。每个推理步骤输出“与偏好/预算/时间对齐的候选或子约束”，给下游 agent 做精确输入。

### 3.2 六个专职 agent

| Agent | 干什么 | 用到的算法 |
|:--|:--|:--|
| 城际交通 | 出发/返程车次选择 | **双向时间优化**：去程按到达时间升序，返程按出发时间降序，最大化目的地停留时长；预算不够时走两级搜索（先时间最优且总价 ≤ 预算，退化为预算可行但时间次优），并给出缺口 \\(\Delta=\min_{\text{total}}-\text{budget}\\) 用于放松约束或与用户协商 |
| 景点推荐 | 产生候选 POI 序列 | 贪心扩展 + **多属性效用**（地理邻近、评分、热度），非短视选择 |
| 路径规划 | 把 POI 序列排成每日行程 | **VRP-TW**（带时间窗的车辆路径）+ OR-Tools 定制求解器 |
| 酒店推荐 | 按行程嵌入住宿 | 距离/房型/预算约束筛选 |
| 餐饮推荐 | 按口味优先、再优化物流 | 偏好优先的两级分配 |
| 行程合成 / 精修验证 | 汇总、校验、反思修复 | 规则校验 + reflection 回炉 |

### 3.3 路径规划：把“日”当“车”

这是我认为设计最巧的一处。把每天的行程视作 VRP-TW 里的一辆车：

$$
\min z=
\underbrace{\sum_{d\in D}\sum_{i\in A}\sum_{j\in A, j\ne i} t_{ij}x_{ijd}}_{\text{总交通时间}}
\;+\;
\underbrace{\sum_{i\in A_c} p_i(1-y_i)}_{\text{跳过可选 POI 的惩罚}}
$$

$$
x_{ijd}=\begin{cases}1,&\text{第 }d\text{ 天走了 }(i,j)\\0,&\text{否则}\end{cases}
\qquad
y_i=\begin{cases}1,&i\text{ 被访问}\\0,&\text{否则}\end{cases}
$$

时间窗与一致性约束：

$$
e_d\le c_{id}\le l_d,\qquad e_i\le c_{id}\le l_i
$$
$$
c_{id}+t_{ij}+s_i=c_{jd},\qquad \min(i)\le s_i\le\max(i)
$$
$$
\sum_{i\in A}y_i\le N
$$

- 用 \\(p_i\\) 惩罚“漏掉非必去 POI”，在**覆盖面**与**满足必去约束**之间做平衡；
- 停留时长 \\(s_i\\) 是**松弛变量**（有上下界），避免“卡死在某点”；
- 细粒度用户约束（“必须在第 2 天去某馆”“这两个点同一天”）通过 OR-Tools 的 `VehicleVar(i)`（取访问日）与 `NextVar(i)`（取后继点）编码进去：

$$
d_i=\mathrm{VehicleVar}(i),\qquad \mathrm{NextVar}(i)=j
$$

这样“经典 VRP-TW 约束 + 用户自定义顺序约束”被压进同一个求解器，不用自己手写搜索。

### 3.4 反思与修复

求解出来的初版行程并不是终点：验证 agent 逐条检查硬约束与环境可行性，把错误步骤回炉重想（reflection），再重跑受影响的那部分。**分模块的前提就是可以局部重算**——这也是多 agent 相对单次生成的最大优势。

## 4. 实验

### 4.1 设置

- **数据**：IJCAI'25 Autonomous Travel Itinerary Planning Challenge 的 ChinaTravel 两阶段数据。Phase 1 有 **1000 条**查询（1–2 个约束的简单查询）；Phase 2 有 **300 条**（easy / medium / hard 三档）。
- **对比**：纯 LLM（Act、ReAct）、神经符号（LLM NeSy、LLM-modulo）；底座涵盖 DeepSeek-V3、GPT-4o、Qwen3-8B、Llama-3.1-8B、Mistral-7B。
- **指标**：EPR-micro / EPR-macro / C-LPR / FPR + DAV / ATT / DDR，加权
  $$
  \text{Overall}=10\%\cdot\text{EPR}_{micro}+10\%\cdot\text{EPR}_{macro}+25\%\cdot\text{C-LPR}+40\%\cdot\text{FPR}+5\%\cdot(\text{DAV}+\text{ATT}+\text{DDR})
  $$

### 4.2 结果

CAMTP 在不同底座上的 Overall 分数（同一套框架）：

| 底座 | Phase 1 | Phase 2 |
|:--|--:|--:|
| DeepSeek-V3 | **77.23** | **84.32** |
| GPT-4o | 70.26 | 80.54 |
| Qwen3-8B（微调后） | 86.73 | 76.66 |

- **硬约束指标（C-LPR / FPR）上优势最大**，这正是 NL2C2DSL + 求解器路线的目标；
- 软偏好指标也不落下风：说明“先保可行、再谈偏好”的设计没有牺牲体验；
- 纯 LLM 方法（Act/ReAct）在硬约束上大面积归零——它们能生成“像行程”的文本，但满足不了逻辑约束；神经符号方法明显更好，但 LLM-modulo 的迭代自修成本高。
- 最终该方案获得 **IJCAI'25 Autonomous Travel Itinerary Planning Challenge 冠军**。

### 4.3 消融：SFT 到底值不值

| 模型 | 零样本 DSL 准确率 | 微调后 |
|:--|--:|--:|
| Qwen3-instruct-8B | 86.0% | **97.8%**（+11.8） |
| Llama3.1-instruct-8B | 66.0% | 62.0%（反而降） |

两点观察：

1. SFT 对 DSL 这类**结构化输出**的提升非常直接（+11.8）；
2. 但**不是所有底座都吃得下**：Llama3.1 微调后反而退步，说明底座本身的“结构化生成倾向”很关键。选底座时要看它在代码/形式化文本上的先验，而不是只看参数量。

## 5. 我从这个项目里学到的

1. **能代码化就不要再过一次 LLM。** C2DSL 改成规则映射，是“减少不确定性”而非“提高模型能力”——工程上往往后者更贵。
2. **中间表示要按“可抽取 + 可校验”设计。** 44 类约束的价值不在分类本身，而在于每一类都能被固定字段表达，从而同时服务于抽取、校验与代码生成。
3. **让 LLM 管语义，让求解器管组合。** 城际交通、景点顺序这些本质是组合优化，交给 VRP-TW 比让模型“算一算”稳得多；模型的职责是理解偏好、生成候选、解释结果。
4. **局部重算的能力来自模块化。** 六个 agent 各自有明确输入输出，验证失败时可以只重跑受影响的那一段，这是 reflection 真正可行的前提。

---

论文 PDF 的 Markdown 转写我放在站点的 `/files/papers/camtp.md`，方便直接检索与引用。

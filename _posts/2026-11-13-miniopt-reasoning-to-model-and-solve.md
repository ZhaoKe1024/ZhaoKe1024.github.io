---
title: "MiniOpt：把“建模+求解”压成一条可验证的思维链，用 3B 打 671B"
date: 2026-11-13 22:30:00 +0800
permalink: /posts/2026/11/miniopt-reasoning-to-model-and-solve/
tags:
  - miniopt
  - llm4opt
  - rlvr
  - grpo
  - post-training
categories:
  - paper-reading
excerpt: '小模型要学会“建模求解”，关键不是喂更多题，而是把任务改成可自动判分的形式：五要素建模写进 think，可执行 Pyomo 写进 answer，用一个规则型 OptReward 做 RLVR，再用 OptGRPO 顶住熵坍缩。'
---

> 论文：**MiniOpt: Reasoning to Model and Solve General Optimization Problems with Limited Resources**（TPAMI draft）
> arXiv：<https://arxiv.org/abs/2606.25832>

## 0. 一句话概括

把“自然语言 → 优化模型 → 求解器代码 → 数值答案”当成**一个端到端可验证的任务**来做 RL：CoT 里必须写出五要素模型，答案里必须给可执行代码，reward 全部由规则自动算。3B 的模型在 8 个 benchmark 上平均 SA 59.65，超过 GPT-5（57.54）与 Gemini-2.5-Pro（57.39）。

## 1. 任务是怎么建模的

回到优化问题的标准形式：

$$
\min_{x\in\mathcal{X}\subseteq\mathbb{R}^D} f(x)\quad
\text{s.t. } G(x)\le c
$$

把它拆成**五要素**：

$$
\mathcal{M} = (\text{Variables},\ \text{Objective},\ \text{Constraints},\ \text{Sets},\ \text{Parameters})
$$

| 要素 | 作用 |
|:--|:--|
| Sets | 决定决策变量与约束族的维度、命名 |
| Parameters | 外生常量：成本、系数、预算、需求 |
| Variables | 变量的定义域与上下界（连续/非负/整数/0-1） |
| Objective | 最小化或最大化的表达式 |
| Constraints | 具名的线性/非线性等式与不等式族，构成 \\(G\\) 与上界 \\(c\\) |

为什么要固定这五要素？因为**优化结构对同类问题是不变的，而数值参数会剧烈变化**。学会结构，才谈得上泛化；只拟合数值，等于把题目背下来。

## 2. 思维链与输出长什么样

输出被两个**强制且可解析**的标签切段：

```text
<think>
## Sets:
  ...（集合定义）
## Parameters:
  ...（参数及其来源）
## Variables:
  ...（决策变量、类型、上下界）
## Objective:
  ...（max/min 表达式）
## Constraints:
  ...（约束族，逐条命名）
## Solver:
  ...（选择的开源求解器 + 理由：问题类型、规模、可行性）
</think>

<answer>
```python
import pyomo.environ as pyo
# 完整 Pyomo 脚本：建模 → 调求解器 → 求解 → 打印最优目标值
```
</answer>
```

要点：

- `<think>` 里不仅建模，还要**选求解器**（按问题类型与求解器特性匹配），这是提升匹配度、避免“用错引擎”的关键；
- `<answer>` 必须是**单个代码块**，且能跑通、能解析出最优目标值 \\(\hat{f}\\)；
- 这样一来，动作空间从“自由文本”塌缩成“可程序化验证的产物”，中间结构与最终答案都能被确定性解析。

这就是所谓 **reasoning-to-model-and-solve paradigm**：思考阶段负责把结构想清楚，回答阶段负责把它变成可执行程序。

## 3. 训练过程：SFT 打底 + 两阶段 RL

```text
Base model
   │
   ├─ Mid-training: SFT warm-up（小规模、类型均衡、保留 unknown 样本）
   │
   ├─ Stage-1 RL: 1,585 道相对简单的题（NL4Opt + ICML Competition 训练集）
   │        目标：学会范式 —— 产出合法 think/answer、可执行代码、合理选求解器
   │        信号主要来自格式分 + 结构分
   │
   └─ Stage-2 RL: 类型均匀 + 场景分布匹配的混合题
            目标：优化泛化 —— 把重点转到准确率分，细抠变量/约束写法与求解器选择
```

- SFT 数据取自 OptMATH-Train-Pyomo 的子集，刻意让**线性与非线性题数量均衡**（非线性不够就用 MILP 补），并保留全部 “unknown” 标签样本以保住难度与多样性；
- 两个 RL 阶段**共用同一套 reward 与优化算法**，只换数据分布。这种“先立范式、再练泛化”的分段，是有限算力下最实用的一招。

## 4. OptReward：三件套 + 一个门

| 分量 | 取值 | 依据 |
|:--|:--|:--|
| 格式分 \\(S_{\mathrm{fmt}}\\) | \\(+1\\) / \\(-1\\) | 必须有且仅有一对 `<think>` 与 `<answer>`，且顺序正确 |
| 五要素分 \\(S_{\mathrm{five}}\\) | \\(0.2\sum_{k=1}^5 I_k\\)（若至少出现 1 个），否则 \\(-1\\) | 检查 `## Sets/Parameters/Variables/Objective/Constraints` 五个标记是否存在 |
| 准确率分 \\(S_{\mathrm{acc}}\\) | \\(+2\\) / \\(-1.5\\) / \\(-2\\) | 执行代码：目标值等于真值 / 不等 / 跑不起来 |

$$
S_{\mathrm{five}}=
\begin{cases}
0.2\sum_{k=1}^{5}I_k, & \sum_{k=1}^{5}I_k\ge 1\\
-1, & \sum_{k=1}^{5}I_k=0
\end{cases}
\qquad
S_{\mathrm{acc}}=
\begin{cases}
+2, & \hat{f}=f^\star\\
-1.5, & \hat{f}\ne f^\star\\
-2, & \text{无法执行}
\end{cases}
$$

**门控**：格式不对，总分直接打到全局最低：

$$
R=
\begin{cases}
-4, & S_{\mathrm{fmt}}=-1\\
S_{\mathrm{fmt}}+S_{\mathrm{five}}+S_{\mathrm{acc}}, & S_{\mathrm{fmt}}=1
\end{cases}
$$

三个设计意图：

1. **格式门**强制模型早期就学会正确输出结构，同时省掉对畸形样本的昂贵执行；
2. **五要素分**是“存在性”规则，不需要标注金标准五元组，避免标注偏差，同时把建模蓝图塑形成可解析的骨架；
3. **准确率分**用目标值等值来认证正确性——这是唯一“真信号”，但它是稀疏的，所以前两件套负责把探索空间收窄。

## 5. OptGRPO：为什么要改 GRPO

在 Stage-2 里模型要探索复杂解题模式，标准 GRPO 有两个副作用：

- **对称裁剪**（\\(1\pm\varepsilon\\)）会让重要性权重上限过早收紧，导致模型**区分不了对错解**就收敛了；
- **KL 惩罚**压制探索。

于是 OptGRPO 做了三件事：

1. **去掉 KL 惩罚**（\\(\beta=0\\)）；
2. **非对称裁剪** \\([1-\varepsilon_{\text{low}},\,1+\varepsilon_{\text{high}}]\\)，且 \\(\varepsilon_{\text{high}} > \varepsilon_{\text{low}}\\)；实训练到 \\(\varepsilon_{\text{high}}=0.28\\)。含义是：**允许概率上调走得更远（鼓励尝试低概率但可能更优的模式），但对概率下调保留硬下界**。之所以敢放松上界，是因为 CoT 的结构已被五要素约束住，rollout 不会野到不可解析；
3. **token 级策略梯度损失**，放大对有效解题模式的奖励信号，避免高质量长轨迹被按序列长度摊薄。

目标函数：

$$
\mathcal{J}_{\text{OptGRPO}}(\theta)=
\mathbb{E}_{q\sim P(Q),\, \{o_i\}\sim\pi_{\theta_{\text{old}}}(\cdot|q)}
\frac{1}{\sum_i |o_i|}
\sum_{i=1}^{G}\sum_{t=1}^{|o_i|}
\min\!\Bigl[\rho_\theta \hat{A}_{i,t},\;
\mathrm{clip}\bigl(\rho_\theta,\,1-\varepsilon_{\text{low}},\,1+\varepsilon_{\text{high}}\bigr)\hat{A}_{i,t}\Bigr]
$$

$$
\rho_\theta=\frac{\pi_\theta(o_{i,t}\mid q,o_{i,<t})}{\pi_{\theta_{\text{old}}}(o_{i,t}\mid q,o_{i,<t})}
$$

## 6. 实验

### 6.1 八个 benchmark 上的主性能

8 个 benchmark 覆盖 7 种优化类型、22 个场景（NL4Opt、ICML Competition、Mamo.Easy、Mamo.Complex、NLP4LP、ComplexOR、IndustryOR、OptiBench）：

| 模型 | Avg. SA | 备注 |
|:--|--:|:--|
| Qwen2.5-14B-Instruct | 47.46 | 通用指令模型 |
| DeepSeek-V3 (671B) | 60.14 | |
| DeepSeek-R1 (671B) | 60.85 | |
| GPT-5 | 57.54 | |
| LLMOPT-14B | 60.10 | 学习型 |
| Step-OPT-7B | 52.22 | 学习型 |
| **MiniOpt-3B** | **59.65** | 高于 GPT-5、Gemini-2.5-Pro（57.39） |
| **MiniOpt-7B** | **64.76** | 全表第一 |

在 Mamo.Complex（最难的子集）上，MiniOpt-3B / 7B 拿到 **47.39 / 51.18**，超过所有通用与思考模型。
参数-性能平面上，MiniOpt 家族处在开权重模型的**经验 Pareto 前沿**：从 3B 到 7B 只涨 5.11 个点，而同族 Step-OPT 从 7B 降到 3B 掉了 12.46 个点。

### 6.2 消融：每个部件值多少

MiniOpt-3B 上的消融（8 benchmark 平均 SA）：

| 配置 | Avg. SA | Δ |
|:--|--:|:--|
| MiniOpt-3B（完整） | **59.65** | — |
| w/o SFT warm-up | 56.60 | −3.05 |
| w/o RL | 27.24 | **−32.41** |
| w/o Two-stage RL | 55.07 | −4.58 |
| w/ Random Selection（代替技能选择） | 58.33 | −1.32 |
| w/o OptReward（只留最终答案信号） | 57.96 | −1.69 |
| w/ 原版 GRPO | 56.54 | −3.11 |

两个读数很重要：

- **RL 是命门**：去掉 RL 直接掉到 27.24，说明单靠 SFT 学不会“自己检查自己”。
- **OptReward 的绝对涨幅不大，但它是不可替代的**：没有它就没有无需 CoT 标注的可验证监督信号；而在最难的 Mamo.Complex 上，去掉它掉 **8.53** 个点——差距全在难例上。

### 6.3 训练过程：entropy 与 reward 的走势

这是我认为最能说明 OptGRPO 价值的一组实验：在 Stage-2 全程记录 reward 与策略熵。

| 现象 | 标准 GRPO | OptGRPO |
|:--|:--|:--|
| reward 收敛值 | 与 OptGRPO 相当 | 与 GRPO 相当 |
| policy entropy | **急剧下滑，早期坍缩** | **全程维持更高水平** |

也就是说：**两者最终 reward 差不多，但 GRPO 是靠“把策略收窄”换来的。** 优化问题的解空间往往非凸、存在多种合法建模范式，熵一崩，模型就锁死在高概率但那不一定最优的几条推理路径上。OptGRPO 保留更高的熵 = 保留了对多种推理轨迹的探索，这被认为是它在难 benchmark 上更强的直接原因。

reward 设计的敏感性也做了（换 accuracy 打分配置）：

| accuracy reward | Δ₁（对−错） | Δ₂（错−崩） | 收敛 reward | 收敛 entropy | SA (%) |
|:--|--:|--:|--:|--:|--:|
| 原始 (2, −1.5, −2) | 3.5 | 0.5 | 2.65 | 0.110 | **59.65** |
| 变体 A (2, 0, −1) | 2.0 | 1.0 | 3.20 | 0.097 | 59.36 |
| 变体 B (1, −1, −1) | 2.0 | 0.0 | 2.35 | 0.101 | 57.40 |

变体 A 的“收敛 reward 更高”是**假象**：错解不再受罚，reward 整体上移，能力并没变强。变体 B 把“错解”和“执行失败”压成同一档（Δ₂=0），掉 2.25 个点 —— 说明 reward 必须保留**层级**：正确 > 算错 > 跑不动。

### 6.4 OptiBench-Separate：我是怎么构造的，为什么要造

现有的优化 benchmark 几乎都把**数值参数直接嵌在题面文本里**。这带来一个混淆：模型答对，可能是真的会建模，也可能只是擅长从文本里抠数字。为了把这两件事拆开，我构造了 **OptiBench-Separate**：

1. 对 OptiBench 每一题，把题面里内嵌的参数表抽成独立的 `./data/*.csv`；
2. 改写题面：只按文件名引用数据并描述**列语义**，不暴露任何数值；
3. 用一个**确定性（非 LLM）预处理脚本**为每题生成 data summary：文件名、列 schema、数据类型、少量样例值（反映量级）、标准 pandas 访问方式；
4. 输入 = 改写后的题面 + data summary。**除了参数的呈现形式，其他一切都与原来一致**（OptiBench 每题参数不到 723 字节，输入长度基本没变）。

推理侧也统一成两阶段协议，对所有对比方法一致：先识别是否需要外部参数文件、抽取文件名与 schema，再生成代码用 `pd.read_csv` 读入并用 `df.loc/df.iloc` 实例化模型。

结果（同一批 604/605 道题、同一评分口径）：

| 模型 | OptiBench | OptiBench-Separate | ΔSA |
|:--|--:|--:|--:|
| DeepSeek-V3 (671B) | 64.13 | 49.09 | **−15.04** |
| LLMOPT-Qwen2.5-14B | 55.83 | 25.62 | −30.21 |
| StepOPT-Qwen2.5-7B | 48.76 | 28.93 | −19.83 |
| StepOPT-Qwen2.5-3B | 40.00 | 16.36 | −23.64 |
| **MiniOpt-3B** | 55.21 | **38.35** | **−16.86** |

三个结论：

1. **连 671B 的通用模型都掉了 15 个点** → 说明“参数的呈现方式”本身就是难度的一部分，优化推理不只依赖数学建模，也依赖数据被怎么组织；
2. **MiniOpt-3B 的退化幅度（−16.86）与 671B 的 DeepSeek-V3（−15.04）相当**，却在绝对准确率上远高于其他学习型模型 → 结构导向学习确实降低了对“参数耦合表示”的依赖；
3. 在所有学习型模型里，MiniOpt 的**退化最小、绝对分最高**。

错误模式分析更扎心：605 题里有 159 题由对变错；其中**主导失败模式（57.9%）不是“没发现外部文件”，而是“正确写了 `pd.read_csv`，却继续用幻觉出来的系数去建约束”**。这暴露了现有优化数据集的一个被忽视的数据工程问题：结构学习与参数抽取被耦合在一起，模型暗中记了数值模式，一旦解耦就露馅。

### 6.5 MIPLIB-NL：参数解耦下的结构可扩展性

MIPLIB-NL 本身就是“问题描述 + 数据文件”分离的格式，正好用来测**结构可扩展性**。推理流程相应改造：先定位外部数据文件与 schema → 从描述重建模型 → 生成代码从文件读入数据求解。

结论：

- **MiniOpt-3B 成功解出含 1,620 个 0-1 决策变量、3,285 条约束的实例**；
- **MiniOpt-7B 解出变量数超过 1,620、约束数 2,520 的实例**；
- 但“最大可解实例”**并不随模型规模单调增长** —— 可解性不只取决于模型大小，还取决于实例的结构复杂度与约束耦合程度。

这条负面结论其实很有价值：它说明参数解耦评测能暴露“上下文长度 vs 结构复杂度”这两种不同瓶颈，而不是把所有失败都归因于模型太小。

## 7. 结论与局限

MiniOpt 的核心不是“更小的模型”，而是**把任务改造成可验证的形式**：五要素 CoT 提供结构，可执行代码提供认证，OptReward 提供低成本监督，OptGRPO 提供探索空间。

局限也很清楚：

- 五要素是“存在性”检查，不保证建模语义完全正确（真正的判别仍靠执行结果）；
- 对超高维问题，参数描述变长会重新变成上下文问题；
- 3B 级模型在指令遵循要求高的子集（如 IndustryOR）仍吃亏。

---

代码与训练细节（含全部 prompt、OptiBench-Separate 构造脚本）都在补充材料里，欢迎来踩坑。

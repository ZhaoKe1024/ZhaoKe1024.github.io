# A Constraint-Aware Multi-Agent Optimization Framework with Robust Domain Specific Language Generation for Travel Planning

**Hong Qian¹·³, Yiyi Zhu¹·², Yitong He¹, Chenxi Li⁴, Yuanhao Liu¹, Linhan Li², Yangde Fu¹, Xiang Shu², Bin Zi², Ke Zhao¹, Gongduo Zhang², Xingyu Lu²⁽ˢ⁾**

¹ Shanghai Institute of AI for Education and School of Computer Science and Technology, East China Normal University, Shanghai 200062, China
² Ant Group, Hangzhou 310023, China
³ Shanghai Innovation Institute, Shanghai 200231, China
⁴ College of Civil Engineering, Shanghai Normal University, Shanghai 200234, China

*Frontiers of Computer Science*, Just Accepted Manuscript. DOI: 10.1007/s11704-026-52005-y
Code: <https://github.com/zhuyiyi-123/CAMTP>

> 说明：本文件是论文 PDF 的 Markdown 转写版本，供检索与复用。公式以 LaTeX 表示；为便于阅读，表格做了合并与简化，**引用/数据请以正式发表版本为准**。

---

## Abstract

Today's LLMs, equipped with advanced reasoning capabilities and the ability to invoke external tools, are giving rise to proactive language agents that can perceive, plan, and act in complex real-world contexts. Travel planning exemplifies this paradigm: it demands not only the construction of a feasible itinerary but also the nuanced alignment with users' explicit constraints and implicit preferences. However, existing methods are hindered by two key challenges: first, directly capturing constraints from user input is unreliable; and second, the fact that LLMs lack built-in spatial reasoning capabilities, making it difficult for them to identify efficient or shortest paths. To bridge these gaps, this paper proposes a constraint-aware multi-agent optimization framework for travel planning. At the core of our approach is NL2C2DSL (Natural Language to Constraints to Domain-Specific Language), a pipeline that leverages LLMs to extract both explicit and implicit user constraints from natural language queries and translates them into a structured domain-specific language (DSL) through pattern-based mapping. Building on this formalized representation, we design a collaborative multi-agent framework comprising specialized agents for intercity transportation, attraction recommendation, path planning, hotel and restaurant selection, itinerary synthesis, and refinement/validation. Through coordinated interaction among specialized agents, our framework jointly constructs a coherent and comprehensive travel plan. Extensive experiments across different phase dataset validate our framework's superior travel planning performance. The proposed framework has won the championship on the Autonomous Travel Itinerary Planning Challenge at IJCAI'25 competition. Our codes are available at https://github.com/zhuyiyi-123/CAMTP.

**Keywords:** travel planning, decision making, constraint optimization, multi-agent, large language model

---

## 1 Introduction

As global tourism grows, efficient travel planning becomes increasingly critical for both individual travelers and the tourism industry. Travel planning is a complex decision-making task that involves organizing destinations, activities, and logistics under multiple constraints. Effective solutions must reconcile spatial, temporal, budgetary limits and user preference. Although advancements have been made in AI-driven planning systems, existing approaches frequently exhibit limitations in long-horizon reasoning, multi-constraint and complex optimization, and joint learning of neural-symbolic systems.

Recent advances in LLMs have revolutionized human-computer interaction by enabling users to accomplish complex tasks through natural language. Despite their superior interactivity, LLMs often lack robust spatial reasoning capabilities, leading to suboptimal route planning efficiency. In our competition, participants are required to convert highly diverse natural language requests into executable travel plans. ChinaTravel addresses this by introducing a Python-like domain-specific language (DSL) that formalizes user requirements and validates them via an abstract syntax tree. Although ChinaTravel has achieved promising results in applying LLMs to travel planning, it still faces two main limitations:

(i) the current NL2DSL pipeline struggles with limited constraint coverage, prompt dependency, and ambiguous user expressions;
(ii) excessive reliance on LLMs which inherently lack spatial perception may also result in the final generated plan lacking robustness, with a considerable likelihood of errors occurring during the generation process.

To bridge these two gaps, this paper proposes a Constraint-Aware Multi-agent optimization framework for Travel Planning (CAMTP) through two key innovations.
To address limitation (i), we employ LLMs to extract constraints from the original queries, then generate DSL through **pattern matching** instead of directly translating natural language queries into DSL, which is challenging due to sparse DSL data and LLMs' weak grasp of formal syntax. To enhance constraint recognition, we fine-tune the LLM by synthetically generated training data, significantly improving its ability to identify and disambiguate implicit requirements. Furthermore, to handle incomplete or ambiguous location mentions, we introduce a retrieval-augmented revision step using BM25 ranking to match extracted locations against a candidate address corpus, ensuring robust and accurate constraint correction without the need for annotated data.

To address limitation (ii), we find that integrating LLMs with multiple efficient agents offers a promising solution, leveraging the strengths of natural language understanding and algorithmic precision. We therefore design a comprehensive multi-agent system comprising an Intercity Transportation Assistant, Attraction Recommendation Assistant, Path Planning Assistant, Hotel and Restaurant Recommendation Assistant, Itinerary Synthesis Assistant, and Refinement and Validation Assistant. The collaboration among these agents effectively supports all key aspects of travel planning, bridging the inherent coverage limitations of LLMs and mitigating the high uncertainty in final planning.

Our approach not only improves the robustness of DSL generation, but also advances the practicality of hybrid LLM-solver systems for complex travel planning tasks. The proposed CAMTP has won the championship on the Autonomous Travel Itinerary Planning Challenge at IJCAI'25 competition.

## 2 Related Work

### 2.1 Travel Planning

LLMs have shown significant intelligence in reasoning, optimization and tool-use, offering the potential of promising planning capability. Travel planning is one of the litmus tests for such issues. Previous works tackle travel planning problems with various ways:

1. **Knowledge augmented methods.** TravelRAG retrieves live POI data; TP-RAG adds temporal awareness, each using retrieval to ground LLMs in real-world travel contexts.
2. **Methods combining LLM with traditional algorithms.** TRIP-PAL translates natural language constraints into formal planning representations so an automated planner can guarantee feasibility; Google Research's approach uses combinatorial optimization to refine LLM-generated itineraries under hard temporal and logistical constraints.
3. **Agent-based modular systems.** TravelAgent and TravelPlanner introduce modular LLM-based agents with tool-use for dynamic recommendation and planning, while ChinaTravel focuses on integrating LLM reasoning with domain-specific knowledge bases.

However, existing methods often treat user constraints as simple textual input, lacking a robust mechanism for reliably extracting and formalizing both explicit and implicit constraints, and they fail to integrate dedicated, efficient spatial reasoning for path optimization. In contrast, CAMTP fully considers both constraints and path planning, and re-executes the solution through a reflection module to rethink erroneous steps.

### 2.2 LLM-based Agents

Mainstream LLM agent methods rely on tool invocation and integrate mechanisms such as self-reflection, long-term planning, and self-correction. Existing research generally follows two dominant paradigms: (i) designing multi-model workflows where different LLMs are assigned specialized roles — often critically dependent on powerful proprietary models and fails to train intrinsic agentic capabilities; (ii) imitation learning from large-scale expert trajectories — effective for skill acquisition but expensive, difficult to scale, and hindering self-improvement through dynamic environmental interaction. CAMTP avoids both by combining a fine-tuned small model (constraint extraction) with symbolic/optimization solvers (planning).

## 3 Preliminaries

### 3.1 Travel Itinerary Planning

Let \(C\) denote a set of cities, \(P\) the set of candidate points-of-interest (POIs) including attractions, restaurants, hotels, and transportation options, and \(Q\) a user query expressed in natural language. A travel planning task seeks to construct a multi-day itinerary

$$
\pi=\langle a_1,a_2,\dots,a_T\rangle,
$$

where each \(a_t\in P\) denotes an activity scheduled at time step \(t\). The solution \(\pi\) must satisfy:

- **Environmental Feasibility:** \(\pi\) is valid under real-world constraints (e.g., POI availability, opening hours, transportation feasibility).
- **Logical Constraints:** \(\pi\vDash\varphi_i\) for all user-specified hard requirements \(\varphi_i\in\mathcal{L}_{DSL}\), such as budget limits or arrival deadlines.
- **Preference Optimization:** \(\pi\) maximizes (or minimizes) soft objectives \(f_j(\pi)\) (e.g., maximizing the number of attractions or minimizing travel time).

Formally, the task is to find

$$
\pi^\star=\arg\max_{\pi\in\mathcal{S}(Q)}\ \mathcal{U}(\pi)
\qquad\text{s.t.}\quad \pi\ \text{is feasible and}\ \pi\vDash\{\varphi_i\}.
$$

Where \(\mathcal{S}(Q)\) is the search space induced by the query \(Q\), and \(\mathcal{U}(\pi)\) is a composite utility function derived from the soft objectives \(\{f_j(\pi)\}\) to reflect user preferences.

### 3.2 Domain-Specific Language

A DSL for travel planning is a formal language \(\mathcal{L}_{DSL}\) over variables \(\mathcal{X}\) (activities), operators \(\mathcal{O}\) (logical, relational, arithmetic), and concept functions \(\mathcal{F}\) (e.g., \(\text{cost}(x)\), \(\text{dist}(x,y)\)). Each expression \(\varphi\in\mathcal{L}_{DSL}\) defines a constraint or preference whose semantics \(\varphi(P)\in\{\text{True},\text{False}\}\) determines whether a plan \(P\) satisfies the requirement.

## 4 Methodology

CAMTP has three core modules:

- **Environment module** — the communication interface between user requests and the trained LLM; realized by SFT on large-scale data plus post-processing, aligned with user queries and environment constraints.
- **Thinking module** — uses the fine-tuned LLM to analyze user inputs and systematically infer task-specific constraints, making implicit requirements explicit.
- **Action module** — a multi-agent framework that coordinates task execution and produces the final solution.

### 4.1 Environment Module

#### 4.1.1 Natural Language to Constraints to Domain-Specific Language (NL2C2DSL)

We build upon the NL2DSL framework of the ChinaTravel benchmark and propose NL2C2DSL. In particular, we enhance the intermediate stage of the three-step NL2DSL process, namely the transformation from natural language into structured constraints (NL2C).

We redesign the representation format of structured constraints and consolidate them into **44 distinct categories**, systematically derived from the ChinaTravel dataset and the competition datasets (tpc phase 1 and tpc phase 2). Under this benchmark setting, these constraint types fully cover user requirements, and this taxonomy is more suitable for extraction by LLMs. Furthermore, the redesigned structured constraints are **directly convertible into DSL format** without additional post-processing.

Each type of structured constraint includes a **return content specification field** and an **output format example field**, which together enforce greater stability and accuracy in LLM outputs.

In the subsequent stage of structured constraint conversion into DSL (**C2DSL**), we leverage the redesigned constraints to enable a **direct code-based transformation**. Compared with the fully prompt-based approach, the code-based method eliminates the token overhead of LLM inference in this stage and significantly improves both accuracy and efficiency.

**Table 1. Structured Constraint List in NL2C2DSL (44 categories, grouped).**

| Group | Constraints |
|:--|:--|
| Format | `start city`, `target city`, `days`, `people number` |
| Budget | `total cost()`, `intercity cost()`, `accommodation cost()`, `restaurant cost()`, `attraction cost()`, `inner city cost()` |
| Intercity / in-city transport | `go must type()`, `go must not type()`, `back must type()`, `back must not type()`, `must inner city transportation()`, `must not inner city transportation()`, `distance transport()` |
| Accommodation | `must not accommodation name()`, `must accommodation name()`, `must accommodation name any()`, `must not accommodation type()`, `must accommodation type()`, `must accommodation type any()`, `must room type()`, `must not room type()`, `distance hotel()` |
| Dining | `must restaurant type()`, `must not restaurant type()`, `must restaurant type any()`, `must restaurant name()`, `must not restaurant name()`, `must restaurant name any()`, `restaurant stay time()` |
| Attraction | `must attraction name()`, `must not attraction name()`, `must attraction name any()`, `must attraction type()`, `must not attraction type()`, `must attraction type any()`, `attraction between()`, `attraction stay time()`, `attraction seq()` |
| Temporal | `activate start time()`, `activate end time()` |
| Others | remaining transport/room-matching and completeness constraints |

#### 4.1.2 Retrieval-Augmented Revising (RAR)

To mitigate validation failures caused by incomplete or ambiguous location mentions, extracted location names are used as queries within a corpus of candidate addresses, ranked by **BM25**:

$$
\mathrm{score}(q,d)=\sum_{t\in q}\mathrm{IDF}(t)\cdot
\frac{f(t,d)\cdot(k_1+1)}{f(t,d)+k_1\cdot\bigl(1-b+b\cdot\frac{|d|}{\mathrm{avgdl}}\bigr)},
$$

where \(f(t,d)\) is the term frequency of token \(t\) in \(d\), \(|d|\) is the address length, \(\mathrm{avgdl}\) is the average address length in the corpus, and \(k_1,b\) are hyper-parameters. The highest-scoring address is selected to correct the constraints. This unsupervised approach resolves validation failures rapidly without annotated training data.

#### 4.1.3 Supervised Fine-Tuning

Using ChinaTravel benchmarks (easy, medium, human, human1000) and the competition datasets (tpc phase 1, tpc phase 2), we construct an NL2C fine-tuning dataset:

- For competition datasets, synthetic SFT data are generated by **sampling and recombining constraints** extracted from these datasets, rather than using original natural-language queries (which are reserved exclusively for evaluation, avoiding overlap).
- The SFT dataset spans 44 constraint categories with a pronounced **long-tailed distribution**, reflecting real imbalance. High-frequency categories are cost-related (550–600 instances each), while fine-grained constraints (e.g., `attraction seq`, `attraction stay time`) are rare.
- For datasets with DSL annotations, a **code-based conversion** transforms DSL annotations into the redesigned structured constraints, which serve as supervision.
- For human1000 (no DSL annotations), we employ **DeepSeek-V3** as the backbone to generate additional structured constraint annotations, maximizing usable SFT data.

To ensure data quality, comprehensive cleaning, filtering, and deduplication are performed across all annotated datasets.

### 4.2 Thinking Module

Given the structured constraints extracted by the NL2C stage, this module systematically analyzes user requests and decomposes them into task-specific requirements. It invokes specialized reasoning processes for different aspects of the trip: intercity transportation selection, attraction recommendation, path planning, restaurant allocation, and hotel arrangement. Each reasoning step outputs structured constraints or candidate options that align with user preferences, budget, and temporal requirements. These intermediate results provide precise guidance for downstream multi-agent execution.

### 4.3 Action Module

#### 4.3.1 Intercity Transportation Agent

The agent dynamically resolves spatiotemporal, budgetary, and preference constraints with a **bidirectional temporal optimization** strategy: outbound options sorted by ascending arrival time; return trips reverse-sorted by departure time to maximize destination stay duration. When financial constraints threaten viability, a **two-tiered search protocol** first pursues temporally optimal pairs with cumulative cost \(\le\) user budget, then defaults to fiscally feasible but temporally suboptimal alternatives. Budgetary shortfalls are quantified by gap analysis

$$
\Delta=\min_{\text{total}}-\text{user budget},
$$

to guide constraint relaxation or user negotiation. Modal constraints (e.g., rail-only transit for specific city pairs) are enforced as context-sensitive policies.

#### 4.3.2 Attraction Recommendation Agent

The agent expands a candidate POI set incrementally. The selection heuristic at each expansion step is **not myopic**: it is governed by a multi-attribute utility function optimizing a composite score of geospatial proximity, public ratings, and popularity. Iterative expansion continues, appending the highest-utility attraction at each step, until the user-defined itinerary length is satisfied.

#### 4.3.3 Path Planning Agent

The agent formalizes POI sequence planning as a **Vehicle Routing Problem with Time Windows (VRP-TW)** and employs a customized solver based on OR-Tools. "Vehicles" are modeled as travel days. The objective is

$$
\min z=\underbrace{\sum_{d\in D}\sum_{i\in A}\sum_{j\in A, j\ne i}t_{ij}x_{ijd}}_{\text{total travel time}}
\;+\;\underbrace{\sum_{i\in A_c}p_i(1-y_i)}_{\text{penalty for skipping optional POIs}},
$$

with binary decisions

$$
x_{ijd}=\begin{cases}1,&\text{route }(i,j)\text{ travelled on day }d\\0,&\text{otherwise}\end{cases},
\qquad
y_i=\begin{cases}1,&i\text{ visited on some day}\\0,&\text{otherwise}\end{cases}.
$$

Note that unlike traditional VRP-TW, it is often infeasible to visit all recommended POIs, so for the set of non-mandatory POIs \(A_c\) we introduce penalty \(p_i\) to balance coverage against must-visit constraints. Travel time satisfies \(t_{ij}=t_{ji}\). Time-window constraints are compiled as

$$
e_d\le c_{id}\le l_d,\qquad e_i\le c_{id}\le l_i,\qquad \forall i\in A,\ \forall d\in D,
$$

where \(c_{id}\) is the arrival time at POI \(i\) on day \(d\), and \([e_i,l_i]\) is its available window; a daily window \([e_d,l_d]\) is added per day. For every edge \((i,j)\) a multi-modal local transport planner (walking, public transit, ride-sharing) generates candidate routes, and route feasibility requires

$$
c_{id}+t_{ij}+s_i=c_{jd},\qquad \forall i,j\in A,\ \forall d\in D,
$$

with stay duration \(s_i\) relaxed and bounded: \(\min(i)\le s_i\le\max(i)\). Each POI is visited at most once:

$$
\sum_{i\in A}y_i\le N.
$$

Fine-grained user constraints are compiled into machine-readable time/priority constraints using OR-Tools primitives:

$$
d_i=\mathrm{VehicleVar}(i),\qquad \mathrm{NextVar}(i)=j,
$$

so that visit-day constraints, same-day/different-day constraints, and consecutive-visiting constraints are all enforced by the solver. This unifies classical VRP-TW constraints with rich user-specific constraints.

#### 4.3.4 Hotel and Restaurant Recommendation Agent

Operating as two specialized agents, it embeds lodging and meal events into the established attraction itinerary. For restaurants, the agent follows a user-centric principle: **prioritize personal taste, then optimize logistics**; hotels are selected subject to distance, room type, and budget constraints relative to the daily route.

#### 4.3.5 Itinerary Synthesis and Refinement/Validation

The synthesis agent assembles the partial plans (transport, attractions, paths, hotels, restaurants) into one coherent multi-day itinerary. The refinement and validation agent checks every hard constraint and environmental feasibility condition, and invokes a **reflection** step to rethink erroneous segments, then re-executes only the affected parts. This modular re-execution is the practical basis for reliable self-correction.

## 5 Experiments

### 5.1 Experimental Setup

**Benchmark and tasks.** We evaluate CAMTP on the benchmark provided by the IJCAI competition: **Phase 1** contains 1,000 queries (relatively simple, 1–2 constraints); **Phase 2** contains 300 queries categorized into easy, medium, and hard levels.

**Compared methods.** Pure LLM-based: Act (direct action sequences), ReAct (alternating reasoning and acting). Neuro-symbolic: LLM NeSy (enhanced DSL-based planning) and LLM-modulo (iterative self-refinement with a symbolic verifier). All methods share base LLMs, including DeepSeek-V3, GPT-4o, Qwen3-8B, Llama-3.1-8B, Mistral-7B.

**Metrics.** Four hard-constraint metrics — Environmental Pass Rate (EPR-micro / EPR-macro), Conditional Logical Pass Rate (C-LPR), Final Pass Rate (FPR) — and three soft-preference metrics — Daily Average Visits (DAV), Average Travel Time (ATT), Daily Dining Recommendations (DDR). The overall score is the weighted sum with weights 0.1, 0.1, 0.25, 0.4, 0.05, 0.05, 0.05:

$$
\text{Overall}=10\%\cdot\text{EPR}_{micro}+10\%\cdot\text{EPR}_{macro}+25\%\cdot\text{C-LPR}+40\%\cdot\text{FPR}
+5\%\cdot\text{DAV}+5\%\cdot\text{ATT}+5\%\cdot\text{DDR}.
$$

**Training setting.** LoRA (rank 64, alpha 128, all linear layers), 3 epochs, cosine decay, initial LR \(2\times10^{-4}\), warmup ratio 0.03.

### 5.2 Results

Overall scores of CAMTP (selected backbones):

| Backbone | Phase 1 Overall | Phase 2 Overall |
|:--|--:|--:|
| DeepSeek-V3 | 77.23 | 84.32 |
| GPT-4o | 70.26 | 80.54 |
| Qwen3-8B (fine-tuned) | 86.73 | 76.66 |

Key observations:

1. CAMTP consistently achieves the highest overall score and best average ranking, validating its effectiveness in complex multi-constraint scenarios, while remaining competitive on soft-preference metrics.
2. Pure LLM baselines (Act, ReAct) produce structurally coherent itineraries but fail badly on strict constraints; since logical metrics (C-LPR, FPR) are defined only for environmentally feasible itineraries, their failures on environmental feasibility cascade into zero scores.
3. Neuro-symbolic approaches (LLM NeSy, LLM-modulo) improve over pure LLM methods by enforcing constraint satisfaction, but LLM-modulo incurs high verification cost, and both depend heavily on the symbolic layer's expressiveness.
4. A method can show non-zero EPR-micro but zero EPR-macro when every generated itinerary violates at least one environmental constraint.

### 5.3 Ablation Study

Figure 2 compares NL-to-DSL performance across model configurations:

| Model | Zero-shot DSL accuracy | After SFT |
|:--|--:|--:|
| Qwen3-instruct-8B | 86.0% | **97.8%** (+11.8) |
| Llama3.1-instruct-8B | 66.0% | 62.0% |

SFT is essential for high-accuracy DSL generation, and Qwen3's architecture/pretraining make it especially suitable for DSL-style reasoning; Llama3.1 benefits less and even degrades.

## 6 Conclusion

We presented our solution to the Autonomous Travel Itinerary Planning Challenge at IJCAI'25. First, we proposed the NL2Constraints framework to accurately extract explicit and implicit constraints from diverse user requests, bridging natural language input with executable DSL representations. Second, we designed a multi-agent orchestration framework that balances multiple objectives such as feasibility, cost, time, and user preferences to generate optimized and reliable travel itineraries. Our method won the championship on the IJCAI competition. The results demonstrate that our hybrid design (LLMs + heuristic solvers + multi-agent) offers a robust, accurate, and generalizable solution for complex travel planning tasks. Future work will explore model efficiency, adaptive user preference learning, and broader integration of LLM-agent frameworks.

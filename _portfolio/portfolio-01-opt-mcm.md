---
title: "OPT-MCM"
excerpt: "开放性的优化问题 benchmark 及 Rubrics 生成演化框架。<br/><a href='https://github.com/ZhaoKe1024/OPT-MCM'>GitHub →</a>"
collection: portfolio
permalink: /projects/opt-mcm/
---

**OPT-MCM** 是一个面向开放式优化问题（open-ended optimization problems）的 benchmark 与评测框架：
它把"生成题目 → 生成 Rubrics → 演化迭代 → 自动评测"做成一条可复现的流水线，
用于衡量大模型在优化建模、求解思路与答案质量上的真实能力。

- 可扩展的题目生成器与难度分层
- Rubrics（评分细则）自动生成 + 演化式修订
- 统一的评测脚本与结果汇总

代码：<https://github.com/ZhaoKe1024/OPT-MCM>

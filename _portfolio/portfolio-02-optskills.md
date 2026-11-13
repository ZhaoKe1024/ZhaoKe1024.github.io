---
title: "OptSkills"
excerpt: "优化问题建模求解的 Skill 生成与学习框架。<br/><a href='https://github.com/fujiwaranoM0kou/OptSkills'>GitHub →</a>"
collection: portfolio
permalink: /projects/optskills/
---

**OptSkills** 关注"把运筹优化问题的建模与求解经验固化为可复用的 skill"：
给定自然语言问题，框架先识别问题原型，再套用对应的建模/求解卡片（LP、MIP、SOCP、QP 等），
最后调用求解器并校验结果。

- 问题原型卡片（problem archetype cards）
- 求解器自动检测与选择（COPT / Gurobi / HiGHS / SCIP …）
- 面向 agent 的工具化输出

代码：<https://github.com/fujiwaranoM0kou/OptSkills>

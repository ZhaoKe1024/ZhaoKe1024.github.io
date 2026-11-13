---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

> 把下面各段的学校、年份、导师替换成你的真实信息即可；结构（Markdown 标题 + 列表）可自由调整。

Education
======
* **Ph.D. in Artificial Intelligence / 人工智能**, East China Normal University (ECNU), Shanghai, 2025 – 2029 (expected)
  * Advisor: Prof. Hong Qian
  * Research: LLMs for Optimization, AI for Education
* **M.S.**, 【学校】, 20XX – 20XX
  * Deep Learning, Representation Learning, Statistical Causal Inference
* **B.S.**, 【学校】, 20XX – 20XX
  * Machine Learning, Evolutionary Computing, Scheduling Optimization

Research Experience
======
* 2025 – now: Ph.D. Researcher, ECNU
  * 开放式优化问题 benchmark 与 Rubrics 生成演化框架（OPT-MCM）
  * 优化问题建模求解的 Skill 生成与学习框架（OptSkills）
* 20XX – 20XX: Master Researcher
  * 解耦表示学习用于咳嗽声音诊断（COUGHVID）
  * 异常声音检测（ASD）训练与评估框架

Skills
======
* **Programming:** Python, PyTorch, C/C++, JavaScript, Bash
* **Optimization:** LP / MIP / SOCP / QP modeling, COPT, Gurobi, HiGHS, SCIP, PuLP, Pyomo
* **ML:** representation learning, audio classification, anomaly detection, metaheuristics
* **Tooling:** Jekyll, Git, Linux, Docker

Publications
======
<ul>{% for post in site.publications reversed %}
  {% include archive-single-cv.html %}
{% endfor %}</ul>

Teaching
======
<ul>{% for post in site.teaching reversed %}
  {% include archive-single-cv.html %}
{% endfor %}</ul>

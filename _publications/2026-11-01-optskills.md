---
title: "OptSkills: Learning Generalizable Optimization Skills from Problem Archetypes via Cluster-Based Distillation"
collection: publications
category: conferences
permalink: /publication/optskills/
excerpt: 'Optimization agents reuse experience at the archetype level: cluster problems in an archetype space, roll out a solver portfolio, and distill reusable skills.'
date: 2026-11-01
venue: 'EMNLP 2026 (Findings)'
paperurl: 'https://arxiv.org/abs/2605.29829'
citation: 'Haochen Yang, Ke Zhao, Mengyuan Ma, Xingyu Lu, Xiangfeng Wang, Hong Qian. (2026). &quot;OptSkills: Learning Generalizable Optimization Skills from Problem Archetypes via Cluster-Based Distillation.&quot; <i>EMNLP 2026 Findings</i>.'
---

**Findings of EMNLP 2026.** OptSkills organizes reusable experience by **problem archetype**
instead of surface text similarity. It projects each problem into an archetype embedding, clusters them
with DBSCAN, rolls out a portfolio of modeling paradigms and solvers per cluster, and distills the
trajectories into skills `s = (m, w, p)` that are refined or expanded as new problems arrive.
It reaches **68.27%** micro-average Pass@1 over five benchmarks (+4.81 over the strongest baseline),
**26.91%** on MIPLIB-NL, and **72.79%** on the OOD NLCO split after skill learning.

- Code: <https://github.com/fujiwaranoM0kou/OptSkills>
- Blog: [OptSkills 精读](/posts/2026/11/optskills-archetype-cluster-distillation/)


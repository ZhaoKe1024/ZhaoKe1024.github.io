---
title: "AGEDR：用属性高斯估计把“性别/病程/咳嗽类型”从潜变量里摘出去"
date: 2026-11-13 22:50:00 +0800
permalink: /posts/2026/11/agedr-attributes-gaussian-disentangled-representation/
tags:
  - agedr
  - disentangled-representation
  - vae
  - medical-audio
  - fairness
categories:
  - paper-reading
excerpt: '医疗音频诊断里，属性（年龄、病程、咳嗽类型）会造成 domain shift 与不公平。AGEDR 让属性走一条独立的 AME 通道，用 KL 把 VAE 潜变量的一段对齐过去，再用互信息下界把这段与其余部分拉开——一段管属性，一段管剩下的信息。'
---

> 论文：**Disentangling Representation using Attributes-based Gaussian Estimation for Medical Sound Diagnosis**（arXiv preprint）
> 代码：<https://github.com/ZhaoKe1024/DisentangledRepr>
> arXiv：<https://arxiv.org/abs/2608.29026>

## 0. 中心思想：把“属性”变成潜空间的一段可解释通道

深度学习在医疗信号上很强，但**不可解释**且**不公平**：年龄、性别、病史、生活方式这些属性的分布偏移（domain shift），会让模型在某类人群上系统性变差。医学上我们恰恰希望模型**只看病理，不看人是谁**。

解耦表示学习（DisenRL）的想法是：把潜向量切成“可解释的语义段”与“其余信息段”，再让两者互不掺和。麻烦在于——**潜向量本身没有先验分布，你没法对它做概率密度估计，也就没法算 KL 与互信息。**

AGEDR 的关键观察是：

> 数据样本里既然带了属性标注，属性信息本来就存在于样本中。**那就给属性也建一个可估计的高斯通道**，让 VAE 潜变量的对应段去对齐它。

于是框架由两块拼成：

1. **AME（Attribute Mapping Embedding）**：把标量属性映射成高斯参数，重参数化得到属性的潜向量 \\(\hat{z}^{\alpha}\\)；
2. **VAE**：从原始音频抽出潜向量 \\(z\\)，按维度切成属性段 \\(z^{\alpha}\\) 与非属性段 \\(z^{\beta}\\)，两段一起送进 decoder 做重建。

之后：

- 用 \\(D_{KL}\\) 把 \\(z^{\alpha}\\) 对齐到 \\(\hat{z}^{\alpha}\\)（让 VAE 真的从数据里学到属性）；
- 用互信息最小化把 \\(z^{\alpha}\\) 与 \\(z^{\beta}\\) 拉开（让两段不重叠）；
- 用 encoder 输出的**均值** \\(\mu\\) 训练分类器（避开重参数化噪声）。

## 1. 模块一：AME 怎么把标量属性变成高斯潜向量

属性有两类，走两条映射：

$$
\bigl(\hat{\mu}^{(i)},\hat{\sigma}^{(i)}\bigr)=
\begin{cases}
\operatorname{est}\bigl(M^{dis}\cdot \mathrm{onehot}(a^i)\bigr), & a\in N_+ \ (\text{类别属性})\\[4pt]
\operatorname{est}\bigl(M^{con}\cdot a^i\bigr), & a\in\mathbb{R}\ (\text{连续属性})
\end{cases},
\quad i=1,\dots,r
$$

其中 \\(M^{dis}\in\mathbb{R}^{n^{(i)}\times c^{(i)}}\\) 是类别查表矩阵（\\(c^{(i)}\\) 为类别数），\\(\mathrm{onehot}(\cdot)\\) 选列；\\(M^{con}\\) 把连续标量变成向量；\\(\operatorname{est}(\cdot)\\) 是两层神经网络，输出高斯参数。

再重参数化采样：

$$
\hat{z}^{(i)}=\hat{\mu}^{(i)}+\hat{\sigma}^{(i)}\odot\epsilon^{(i)},\qquad
\epsilon^{(i)}\sim\mathcal{N}(0,1)
$$

**为什么这一步重要？** 直接把属性 one-hot 拼进潜向量（InfoGAN 式做法）只有“离散标签”的信息量，没有分布；而 AME 输出的是**高斯分布参数**，于是后面可以算：

- 与 VAE 潜变量段的 \\(D_{KL}\\)（对齐）；
- 段的互信息（解耦）。

这就是为什么必须先“估成高斯”，否则整条信息论链路断掉。

### 维度怎么切

设 VAE 潜向量维度为 \\(N\\)，切成 \\(r\\) 个属性段加 1 个非属性段：

$$
\sum_{i=0}^{r}n^{(i)}=N,\qquad
z^{\alpha}=\mathrm{concat}\bigl(z^{(1)},z^{(2)},\dots,z^{(r)}\bigr),\qquad
z=\mathrm{concat}\bigl(z^{\beta},z^{\alpha}\bigr)
$$

在 COUGHVID 实验里：\\(N=30\\)，`cough_type`（3 类）占 6 维，`severity`（4 类）占 8 维，剩下 16 维是 \\(z^{\beta}\\)。

## 2. 模块二：对齐——用 KL 把属性“灌进”VAE

$$
L_{attri}^{(i)}=\sum_{i=1}^{r}D_{KL}\bigl(z^{(i)}\Vert\hat{z}^{(i)}\bigr)
=\sum_{i=1}^{r}\log\frac{\hat{\sigma}^{(i)}}{\sigma^{(i)}}
+\sum_{i=1}^{r}\frac{(\sigma^{(i)})^{2}+(\mu^{(i)}-\hat{\mu}^{(i)})^{2}}{2(\hat{\sigma}^{(i)})^{2}}
-\frac{r}{2}
$$

它是 VAE 潜变量第 \\(i\\) 段 \\(z^{(i)}\sim\mathcal{N}(\mu^{(i)},\sigma^{(i)})\\) 与属性通道 \\(\hat{z}^{(i)}\sim\mathcal{N}(\hat{\mu}^{(i)},\hat{\sigma}^{(i)})\\) 之间的 KL。

**效果**：优化这个项会让 VAE 的那一段去逼近 AME 给出的分布。也就是说，**VAE 不需要额外的属性标注参与重建，就能在潜空间里“长”出属性语义**——训练好之后，属性段自动带上了 cough_type / severity 的信息。

## 3. 模块三：解耦——互信息最小化

到这里，\\(z^{\alpha}\\) 与 \\(z^{\beta}\\) 只是**名义上**分开了：输入 \\(X\\) 含全部属性信息，两段之间必然有信息重叠。

用互信息衡量重叠：

$$
MI(z^{\alpha},z^{\beta})=MI(z^{\alpha},x)+MI(z^{\beta},x)-MI(z,x)
$$

但联合分布未知、且各属性段维度 \\(n^{(i)}\\) 不同，无法直接算 KL 当正则项。于是用 **ELBO 给互信息一个上界**（可优化）：

$$
MI(z^{\alpha},z^{\beta})\le L_{Disen}
=\mathbb{E}_{x\sim p(x)}\mathbb{E}_{x'\sim p(x)}KL\bigl(p(z^{\alpha}|x)\,\Vert\,p(z^{\alpha}|x')\bigr)
$$
$$
+\ \mathbb{E}_{x\sim p(x)}\mathbb{E}_{x'\sim p(x)}KL\bigl(p(z^{\beta}|x)\,\Vert\,p(z^{\beta}|x')\bigr)
-\ \mathbb{E}_{x\sim p(x)}\mathbb{E}_{z^{\alpha},z^{\beta}\sim p(z|x)}\log q(x|z)
$$

三项的含义：

| 项 | 含义 | 直觉 |
|:--|:--|:--|
| 第 1 项 | 不同样本 \\(x,x'\\) 导出的 \\(z^{\alpha}\\) 分布之间的 KL | 压掉“哪个样本”带来的差异 → 减少 \\(z^{\alpha}\\) 对样本的可辨识信息 |
| 第 2 项 | 同理作用于 \\(z^{\beta}\\) | 同上 |
| 第 3 项 | 重建项 \\(-\log q(x\|z)\\) | 保底：别把信息压没了，重建还得能还原 |

最小化 \\(L_{Disen}\\) 就是**最小化互信息的上界**：让两段各自只编码自己该管的信息。

## 4. 模块四：分类用 \\(\mu\\)，不用采样

重参数化会往潜向量里注入噪声 \\(\epsilon\sim\mathcal{N}(0,1)\\)——对生成有益，对**预测有害**（噪声会让分类边界抖动）。所以 AGEDR 在分类阶段**直接取 encoder 输出的均值参数 \\(\mu\\)**，丢掉随机性：

$$
\hat{y}=f_{cls}(\mu),\qquad \mu=\text{Encoder}_{\mu}(X)
$$

分类损失用 **focal loss**（COUGHVID 里健康/COVID 比例约 2.68:1，类别不平衡）。

总损失四项加权：

$$
L_{total}=w_1L_{cls}+w_2L_{vae}+w_3\sum_{i=1}^{r}L_{attri}^{(i)}-w_4L_{Disen}
$$

展开 \\(L_{Disen}\\) 后：

$$
L_{total}=w_1L_{cls}+w_2L_{vae}+w_3\sum_{i=1}^{r}L_{attri}^{(i)}
-w_4v_1MI(z^{\alpha},x)-w_4v_2MI(z^{\beta},x)+w_4v_3MI(z,x)
$$

实训练权重：\\(w_1,w_2,w_3,w_4 = 2.0,\,0.3,\,0.0025,\,0.01\\)；\\(v_1,v_2,v_3=1.0,\,1.25,\,5.0\\)。

注意 \\(w_3,w_4\\) 都很小：**对齐与解耦是“轻推”**，不能压过失真与分类主目标；而 \\(v_3=5.0\\) 压 \\(MI(z,x)\\)，是在防止潜变量把所有信息都吞进来。

## 5. 实验

### 5.1 数据

COUGHVID：34,434 条咳嗽音频（众包 + 2,800+ 专家标注）。清洗四步：保留有专家标注的列 → 滤掉无 `status` 的 → `cough_detected < 0.8` 丢弃 → `quality != good` 丢弃。剩 **720** 条；按静音切段并丢弃 < 0.36s → **2,850** 段；统一到 32,360 采样点 @ 22,050 Hz（1.46s）。最终 **2,076 healthy / 774 covid-19**；log-Mel 谱 64×128；属性用 `cough_type`（3 类）与 `severity`（4 类）。

### 5.2 分类性能

| 模型 | Precision | Recall | AUC |
|:--|--:|--:|--:|
| MobileNetV2 | 88.42 | 84.68 | 86.51 |
| AST | 91.32 | 82.24 | 86.54 |
| DisenIB | 90.12 | 91.58 | 90.84 |
| IDB-SR | 94.23 | 92.43 | 93.32 |
| AGEDR（NN, \\(z^\beta+z^\alpha\\)） | 92.52 | **99.00** | 95.50 |
| **AGEDR（SVM, \\(z^\beta+z^\alpha\\)）** | **97.23** | 96.65 | **97.06** |

三点观察：

1. 优于常规分类器（MobileNetV2 / AST）与同类 DisenRL 方法（DisenIB / IDB-SR）；
2. **Recall 特别高（99%）**：在医疗筛查场景里，“宁可多报不可漏报”的方向是对的；
3. 分类头换成 SVM 反而更好（AUC 97.06），说明**潜空间本身已经线性可分**——这恰恰是解耦成功的旁证。

### 5.3 消融一：只用 \\(z^{\beta}\\) 会怎样

| 输入 | Precision | Recall | AUC |
|:--|--:|--:|--:|
| \\(z^\beta+z^\alpha\\) | 92.52 | **99.00** | 95.50 |
| 只用 \\(z^{\beta}\\) | 91.45 | 75.87 | 84.02 |

Recall 从 99% 掉到 75.87%。这证明两件事：

1. **AME 携带的信息里有与域偏移相关的部分**（去掉就明显掉分）；
2. **\\(L_{Disen}\\) 真的把两段拉开了**——如果没拉开，\\(z^{\beta}\\) 里应该还残留属性信息，掉分不会这么明显。

### 5.4 消融二：公平性（t-SNE）

把 \\(z=\{z^\beta,z^\alpha\}\\) 降到二维：

- 按**疾病标签**着色 → 类别清楚分开（模型确实学到了病理）；
- 按 `cough_type`、`severity` 着色 → **均匀撒在整个空间，没有聚类趋势**。

也就是说，分类结果不依赖这些敏感属性，**属性被“脱敏”了**。这既是公平性证据，也是可解释性证据：潜空间里“哪一段管什么”是可验证的。

## 6. 我的复盘：这套设计的取舍

1. **“给属性单独开一条可估计的通道”是核心 trick。** 没有它，KL/MI 都算不出来，解耦只能靠经验性的对抗或正交约束。
2. **KL 项与 MI 项的方向相反但互补**：KL 让属性段**吸收**属性信息，MI 让属性段**不吸收**别的信息。这两条一起才叫解耦，少一条都不行。
3. **权重很小是有意的**：解耦是正则，不是主任务。硬拉会把重建与分类都拉坏。
4. **分类用 \\(\mu\\) 而非采样**这个细节，在生成模型改判别模型时特别容易忘，值得单独记一笔。
5. **局限**：属性是**预定义**的；属性类别多时 AME 的维度分配需要人工设计（本例 6+8+16）；互信息上界的估计仍有方差。

---

代码已经开源（<https://github.com/ZhaoKe1024/DisentangledRepr>），数据清洗脚本与 370 epoch 的训练曲线都在里面。

---
title: "VAE (变分自编码器) 学习笔记"
hideSummary: true
date: 2024-08-19T17:49:24+08:00
draft: true
tags: ["AI", "VAE"]
series: ["AI", "VAE"]
author: ["xubinh"]
type: posts
math: true
---

## 引言

变分自编码器 (Variational AutoEncoder, VAE) 于 2013 年提出, 目的是为了克服传统的自编码器 (AutoEncoder) 在生成任务中的局限性. 自编码器是一种能够学习数据的低维表示形式的神经网络结构, 通常由一个编码器和一个解码器组成. 其中编码器将输入数据压缩为低维的潜在变量, 即将输入数据 $x$ 映射到潜在空间中的一个固定的点 $z$ 上; 解码器则试图由这个潜在变量重建出原始输入, 即将潜在变量 $z$ 映射回输入空间中的一个点 $x'$. 自编码器的目标是训练出这样的编码器-解码器组合, 其中编码器能够提取任意图像的特征, 而解码器能够从任意特征中重建出对应的图像. 实现这一目标的必要条件 (或者说从这一目标可以得到的推论) 是编码器和解码器的复合几乎相当于全等映射, 而编码器和解码器的复和为全等映射的必要条件又是对于每个输入样本, 自编码器均能重建出几乎相同的样本, 因此自编码器使用的是训练集中样本的原始图像与重建图像之间的距离作为损失函数对编/解码器的效果进行评估. 仅仅有评估方法还不够, 还需要确保自编码器确实能够朝着目标进行迭代, 为此自编码器实际上无需做出任何额外的努力——由于从输入空间到潜在空间的过程中存在强制性的信息损失 (维度减少), 编码器天然就更可能朝着更强的特征提取能力的方向进行迭代, 解码器也天然更可能朝着具有更强的图像重建的能力的方向进行迭代 (至少对于训练集中的样本而言).

尽管自编码器通过无监督地学习数据空间与潜在变量空间之间的映射来有效地对输入数据进行压缩并对潜在变量进行重建, 但这种压缩和重建均是建立在确定性的映射的基础上, 无法用于生成任务. VAE 继承了自编码器的设计哲学并将其推广至生成场景下, 其对自编码器框架做出的最大改进就是结合了**变分推断** (Variational Inference) ("变分自编码器" 中的 "变分" 指的就是变分推断). 变分推断试图解决的问题是在一个参数化的概率函数族中如何逼近某个目标概率分布函数, 为此提出了**证据下界** (Evidence Lower Bound, ELBO) 并将原来关于函数族的优化问题转化为关于 ELBO 优化问题. ELBO 可重写为一个 KL 散度项 (即正则化项) 与一个重建误差项的和的形式. 在 VAE 给出的假设下, ELBO 中的 KL 散度项能够转化为可精确计算的闭合形式, 同时 VAE 还使用**重参数化** (Reparameterization) 方法解决了 ELBO 中的重建误差项的梯度传递问题, 从而顺利将变分推断移植到了自编码器框架下. 实验证明 VAE 能够十分出色地完成图像的生成任务, 生成的图像更加逼近原始图像数据.

下面将分小节讨论变分推断, KL 散度, 以及 ELBO 中用于对重建误差项进行近似的蒙特卡洛方法等几个概念. 首先详细介绍一下变分推断与 ELBO.

## 变分推断与 ELBO

谈到变分推断首先需要了解一下贝叶斯推断. 贝叶斯推断 (Bayesian inference) 通过结合以一个先验概率分布的形式存在的先验知识与贝叶斯定理来估计一个后验概率.

1. **贝叶斯推断**: 在贝叶斯统计中, 我们通常对给定观测数据 $x$ 的潜在变量 $z$ 的后验分布感兴趣, 记作 $p(z|x)$.

   $$
   p(z|x) = \frac{p(x|z)p(z)}{p(x)}
   $$

   其中 $p(x|z)$ 是似然函数, $p(z)$ 是先验分布, 而 $p(x)$ 是边际似然或证据.

2. **不可解性**: 对于复杂模型, 直接计算后验分布 $p(z|x)$ 是不可解的, 因为这需要评估边际似然 $p(x)$, 这涉及对所有可能的潜在变量值进行积分:

   $$
   p(x) = \int p(x|z)p(z) \, dz
   $$

**变分推断**是贝叶斯推断的一种近似方法. 它通过将复杂的后验分布 $p*\theta(z|x)$ 用一个更简单的分布 $q*\phi(z|x)$ 来近似, 并通过优化使两者之间的差异 (通常通过KL散度衡量) 最小化. 为什么要额外设置一个 $q_\phi$? 而不是让 $p_{\theta}$ 既生成编码器又生成解码器? 因为不可能使用一个精度有限内存有限的模型去拟合任意复杂的分布, 要将一个概率分布使用神经网络进行近似首先要对概率分布的形式作出假设, 这里解码器 $p_{\theta}(x | z)$ 首先约定为高斯分布, 然后潜在变量的先验分布 $p_{\theta}(z)$ 也约定为标准高斯分布, 于是整个 $p_{\theta}(x)$ 的复杂度最后全部转移到理论的编码器 $p_{\theta}(z | x)$ 里去了, 没法用神经网络对其进行参数化, 必须另外设置一个实际的编码器 $q_{\phi}(z | x)$ 来近似它, 也就是 "让神经网络近似神经网络". 由于变成了近似, $q_{\phi}(z | x)$ 便能够同样被放宽为 (协方差矩阵为对角阵的) 高斯分布.

- 变分推断 (Variational Inference):

  - 首先记住: 提到变分推断就是 ELBO, 因为 ELBO 就是变分推断要优化的目标函数.
  - 这个名词来源于一个称为 "calculus of variations" 的领域, 这个领域所要解决的问题是对泛函 (将函数映射至实数的函数) 进行优化. "Variational" 一词指的就是在一个函数族中进行优化并寻找最优函数 (在概率论领域下即寻找对目标分布的最优近似) 的过程. 变分法 (calculus of variations) (或变分计算 (variational calculus)) 是数学分析的一个领域, 它使用函数和泛函中的微小变化来寻找泛函的极大值和极小值. 泛函通常表示为涉及函数及其导数的定积分. 可以通过变分法中的欧拉-拉格朗日方程找到泛函的最优解.
  - 变分推断的目的是通过优化泛函找到一个最优的近似后验 $q(z|x) \approx p(z|x)$.
  - 在 DDPM 这篇论文中, ELBO 就是要优化的泛函 (因为它将近似后验映射至损失函数值).
  - 由 Jensen 不等式可推出 ELBO.

1. **近似后验**: 变分推断引入一个来自较简单分布族的近似后验分布 $q(z|x)$, 在计算上更易处理. 目标是使 $q(z|x)$ 尽可能接近真实的后验分布 $p(z|x)$.

2. **优化目标**: 近似后验 $q(z|x)$ 和真实后验 $p(z|x)$ 之间的接近程度通过 Kullback-Leibler (KL) 散度来衡量:

   $$
   \mathrm{KL}(q(z|x) \| p(z|x)) = \int q(z|x) \log \frac{q(z|x)}{p(z|x)} \, dz
   $$

   然而, 直接最小化这个 KL 散度也是不可解的, 因为它涉及真实后验分布.

3. **证据下界 (ELBO)**: 取而代之的是, 我们最大化证据下界 (ELBO), 这是一种可解的目标函数, 提供了对对数边际似然 $\log p(x)$ 的下界:

   $$
   \log p(x) \geq \mathbb{E}_{q(z|x)}[\log p(x, z) - \log q(z|x)]
   $$

   ELBO 可以重写为:

   $$
   \mathbb{E}_{q(z|x)}[\log p(x|z)] - \mathrm{KL}(q(z|x) \| p(z))
   $$

   其中, 第一个项是期望对数似然 (重构项), 第二个项是近似后验和先验之间的 KL 散度.

4. **优化**: 通过最大化 ELBO, 我们同时:
   - 鼓励近似后验 $q(z|x)$ 生成能够很好解释观测数据 $x$ 的潜在变量 $z$ (通过期望对数似然项).
   - 确保 $q(z|x)$ 保持接近先验分布 $p(z)$ (通过 KL 散度项).

- ELBO 可以转化为如下形式:

  $$
  \mathbb{E}_{q(z|x)}[\log p(x|z)] - \mathrm{KL}(q(z|x) \| p(z)),
  $$

  其中第一项是对数似然的期望 (即重建项), 而第二项是近似后验与先验之间的 KL 散度.

**证据下界** (ELBO, Evidence Lower Bound) 是变分推断中的一个关键概念. ELBO是对对数边际似然的下界, 通过最大化ELBO, 我们能够优化变分推断的目标, 使得近似分布 $q_\phi(z|x)$ 尽可能接近真实后验分布.

ELBO可以被拆分为两部分: 重建损失和KL散度. 这是由于以下原因:

- **重建损失**: 对应于 $\mathbb{E}_{q_\phi(z|x)}[\log p_\theta(x|z)]$, 衡量的是根据潜在变量 $z$ 重建输入数据 $x$ 的能力. 因此, 它被称为重建损失.

- **KL散度**: 对应于 $D*{KL}(q*\phi(z|x) || p*\theta(z))$, 衡量的是近似分布 $q*\phi(z|x)$ 与先验分布 $p_\theta(z)$ 之间的差异.

这种拆分使得优化过程更为直观, 其中重建损失保证生成样本的质量, KL散度保证潜在空间的结构合理.

## KL 散度

KL散度 (Kullback-Leibler Divergence) 是一种衡量两个概率分布之间差异的非对称度量. 对于两个分布 $P$ 和 $Q$, 其KL散度定义为:

$$
D_{KL}(P || Q) = \sum_i P(i) \log \frac{P(i)}{Q(i)}
$$

在 VAE 中, KL散度用于衡量近似后验分布 $q_\phi(z|x)$ 与先验分布 $p_\theta(z)$ 之间的差异. 最小化KL散度有助于使得近似分布与先验分布尽可能相似, 从而保持潜在空间的结构.

聊到 KL 散度就不可避免会碰到信息熵和交叉熵等等前置概念. 出于完整性考虑, 下面分别给出信息熵, 交叉熵, 以及 KL 散度的一些性质及其证明.

### 信息熵

在信息论中, 一个随机变量的熵 (entropy) 衡量了这个随机变量所蕴含的信息量, 或不确定性. 对于一个服从离散分布 $P: \mathcal{X} \mapsto [0, 1]$ 的随机变量 $X$ 而言, 它的熵定义为

$$
\operatorname{H}(X) \triangleq \mathbb{E}_{x \sim P(\cdot)}\Big[\log\frac{1}{P(x)}\Big] = \sum_{x \in \mathcal{X}} P(x) \log \frac{1}{P(x)} = -\sum_{x \in \mathcal{X}} P(x) \log P(x).
$$

信息论的核心思想是, 所传达信息的 "信息价值" 取决于信息内容的令人惊讶程度. 如果发生极有可能的事件, 则该信息包含的信息量非常小. 另一方面, 如果发生极不可能的事件, 则该信息包含的信息量要大得多. 例如, 知道某个特定数字不会是彩票的中奖号码所提供的信息量非常小, 因为任何特定的数字几乎肯定不会中奖. 但是, 知道某个特定数字会中奖则具有很高的信息价值, 因为它传达了一个概率非常低的事件的发生. 一个事件 $A$ 的信息内容 (information content), 或称自信息 (self-information), 定义为

$$
I(P(A)) \triangleq -\log{P(A)},
$$

其中 $I$ 被称作信息函数 (information function). 从上述定义中可以看出事件 $A$​ 的信息内容与其发生的概率呈反比关系.

- 信息函数 $I$ 需要满足几个性质:

  1. $I$ 是单调递减的. 事件的概率越大其包含的信息越少.
  1. $I(1) = 0$. 一个必然发生的事件不传递任何信息.
  1. $I(P(A) \cdot P(B)) = I(P(A)) + I(P(B))$. 一个由两个独立事件构成的事件的信息内容为这两个独立事件的信息内容之和.

  而对数函数 $\log$​ 是唯一符合上述性质的函数类型.

一个随机变量的信息熵也可表示为其信息内容的期望,

$$
\operatorname H(X) = \mathbb{E}_{x \sim P}[-\log(P(x))] = \mathbb{E}_{x \sim P}[I(P(x))].
$$

### 交叉熵

交叉熵是信息论中的一个度量, 用于量化两个概率分布之间的差异. 它在机器学习中广泛使用, 特别是在分类问题中, 用于定义损失函数. 对于两个离散的概率分布, 真实分布 $P$ 和估计分布 $Q$ 在同一个变量上的交叉熵定义为:

$$
H(P, Q) \triangleq \mathbb{E}_{x \sim P}\Big[\log\frac{1}{Q(x)}\Big] = \sum_{x \in \mathcal{X}} P(x) \log \frac{1}{Q(x)} = -\sum_{x \in \mathcal{X}} P(x) \log Q(x).
$$

其中:

- $P(x)$ 是事件 $x$ 的真实概率,
- $Q(x)$ 是事件 $x$ 的估计概率,
- 求和是对样本空间中所有可能事件进行.

在机器学习的上下文中, $P$ 通常表示真实标签分布 (例如, 分类任务中的独热编码向量), 而 $Q$ 表示模型预测的概率分布.

交叉熵在分类问题中用作损失函数, 鼓励模型预测与真实分布相匹配的概率分布.

### KL 散度

KL 散度, 全称为 Kullback–Leibler 散度 (此外还被称为 "相对熵"), 衡量的是两个概率分布 $P$ 和 $Q$ 之间的差异程度. 一般情况下 $P$ 的角色是真实分布, 而 $Q$ 的角色则是对分布 $P$​ 的近似.

KL 散度常用于变分自编码器 (VAE), 强化学习以及其他需要衡量两个概率分布之间的差异的场景.

对于离散的概率分布 $P$ 和 $Q$, KL 散度的定义为:

$$
D_{\text{KL}}(P(x) \parallel Q(x)) \triangleq \mathbb{E}_{x \sim P}\Big[\log\frac{P(x)}{Q(x)}\Big] = \sum_{x \in \mathcal{X}} P(x) \log \frac{P(x)}{Q(x)} = H(P, Q) - H(P).
$$

KL 散度具有若干性质:

- 非负性: $D_{\text{KL}}(P \| Q) \geq 0$, 取等号当且仅当 $P \equiv Q$.

- 非对称性: $D_{\text{KL}}(P \| Q) \not\equiv D_{\text{KL}}(Q \| P)$,

- When both $P$ and $Q$ are Gaussian distributions, the KL divergence between them has a closed-form solution. For two multivariate Gaussian distributions $\mathcal{N}(\mu_0, \Sigma_0)$ and $\mathcal{N}(\mu_1, \Sigma_1)$, the KL divergence is:

  $$
  D_{KL}(\mathcal{N}(\mu_0, \Sigma_0) \| \mathcal{N}(\mu_1, \Sigma_1)) = \frac{1}{2} \left( \mathrm{tr}(\Sigma_1^{-1} \Sigma_0) + (\mu_1 - \mu_0)^T \Sigma_1^{-1} (\mu_1 - \mu_0) - k + \log \frac{\det \Sigma_1}{\det \Sigma_0} \right)
  $$

  where $k$ is the dimensionality of the distributions.

#### 非负性的证明

易知对于任意 $x > 0$, 有不等式 $\ln x \leq x - 1$ 成立. 于是在离散情况下有

$$
\begin{align*}
D_{\text{KL}}(P(x) \Vert Q(x)) &= \sum_{x \in \mathcal{X}} P(x) \log \bigg(\frac{P(x)}{Q(x)}\bigg)\\
&= -\Bigg(\sum_{x \in \mathcal{X}} P(x) \log \bigg(\frac{Q(x)}{P(x)}\bigg)\Bigg)\\
&\geq -\Bigg(\sum_{x \in \mathcal{X}} P(x) \cdot \bigg(\frac{Q(x)}{P(x)} - 1\bigg)\Bigg)\\
&= -\sum_{x \in \mathcal{X}} P(x) \frac{Q(x)}{P(x)} + \sum_{x \in \mathcal{X}} P(x)\\
&= -1 + 1\\
&= 0,
\end{align*}
$$

即

$$
D_{\text{KL}}(P(x) \Vert Q(x)) \geq 0.
$$

#### Gibbs 不等式

假设存在两个离散概率分布 $P$ 和 $Q$, Gibbs 不等式为

$$
-\sum_{x \in \mathcal{X}} P(x) \log P(x) \leq -\sum_{x \in \mathcal{X}} P(x)\log Q(x),
$$

或等价的,

$$
H(P) \leq H(P, Q),
$$

即概率分布 $P$ 的信息熵总是小于等于其与另一个概率分布 $Q$ 的交叉熵.

Gibbs 不等式的证明也很简单, 直接由 $D_{\text{KL}}(P(x) \parallel Q(x)) = H(P, Q) - H(P)$ 并结合 KL 散度的非负性即可得证.

#### 与交叉熵之间的关系

交叉熵和 KL 散度关系密切. 两个分布之间的交叉熵可以分解为真实分布 $P$ 的熵和 $P$ 到 $Q$ 的 KL 散度之和:

$$
H(P, Q) = H(P) + D_{\text{KL}}(P \| Q)
$$

其中 $H(P)$ 是 $P$ 的熵, 定义为:

$$
H(P) = -\sum_{x} P(x) \log P(x)
$$

这种关系表明, 在机器学习的上下文中最小化分布 $P$ 和 $Q$ 之间的交叉熵 (其中 $P$ 固定为真实分布) 等同于最小化它们之间的 KL 散度. 本质上, 最小化分类任务中的交叉熵旨在调整 $Q$ (模型的预测) 使其尽可能接近 $P$ (真实标签), 从而最小化信息丢失.

## 蒙特卡洛方法

**蒙特卡洛方法**是一类通过随机采样来估计数学表达式的数值结果的计算方法. 在VAE中, 蒙特卡洛方法常用于近似ELBO中的期望值计算.

蒙特卡洛方法 (Monte Carlo methods) 指一类算法, 这类算法的思想是为所希望求得的目标数值精心设计试验, 通过重复多次随机抽样所得到的均值近似该试验的期望并间接近似目标数值. 蒙特卡洛方法的名称来源于摩纳哥 (Monaco) 的一家名为 Monte Carlo Casino 的赌场. 该方法的创始者, 物理学家 Stanislaw Ulam, 借鉴了他叔叔在这家赌场里赌博时的习惯从而发明了蒙特卡洛方法.

最经典的一个说明蒙特卡洛方法的哲学思想的例子是蒙特卡洛积分, 即通过随机撒点来计算 $\pi$ 的值:

|                                                          Monte Carlo method applied to approximating the value of $\pi$                                                          |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| ![Monte Carlo method applied to approximating the value of $\pi$](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d4/Pi_monte_carlo_all.gif/290px-Pi_monte_carlo_all.gif) |

这个例子中蒙特卡洛方法通过重复多次随机撒点并计算点落在扇区内的频率来近似期望 $\frac{\pi}{4}$, 从而间接近似所希望求得的值, 即 $\pi$.

蒙特卡洛方法的重点在于如何设计试验使得该试验的期望与所希望求得的值之间存在能够进行转换的数值关系. 蒙特卡洛方法的理论保证是[柯尔莫哥洛夫强大数定律](https://en.wikipedia.org/wiki/Law_of_large_numbers#Strong_law). 柯尔莫哥洛夫强大数定律的内容是独立同分布的随机变量的均值将以概率 1 趋近于期望.

## VAE

VAE 想要求解最优化问题

$$
\newcommand\dbsp{\ \ }
\newcommand\argmax[1]{\underset{#1}{\operatorname{arg max}}\ }
\newcommand\argmin[1]{\underset{#1}{\operatorname{arg min}}\ }
\begin{equation}
\min_\theta \dbsp -\ln{p_{\theta}(x^{(1)}, x^{(2)}, \dots, x^{(N)})} = -\sum_{i = 1}^N \ln{p_{\theta}(x^{(i)})},
\end{equation}
$$

即尽可能最大化数据集 $X = \\\{x^{(i)}\\\}_{i = 1}^N$ 的似然. 单点对数似然 [TODO]

$$
\begin{equation}
\label{eq:likelyhood}
\begin{aligned}
\ln{p_\theta(x)} &= \ln{p_\theta(x)} \int_z q_\phi(z | x) dz \\
&= \int_z \ln{\left(\frac{p_\theta(x, z)}{p_\theta(z | x)}\right)} q_\phi(z | x) dz \\
&= \int_z \ln{\left(\frac{p_\theta(x, z)}{q_\phi(z | x)} \cdot \frac{q_\phi(z | x)}{p_\theta(z | x)}\right)} q_\phi(z | x) dz \\
&= \int_z \ln{\left(\frac{p_\theta(x, z)}{q_\phi(z | x)}\right)} q_\phi(z | x) dz + \int_z \ln{\left(\frac{q_\phi(z | x)}{p_\theta(z | x)}\right)} q_\phi(z | x) dz \\
&\triangleq L(\theta, \phi; x) + D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z | x)),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:likelyhood2}
\ln{p_\theta(x)} \triangleq L(\theta, \phi; x) + D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z | x)),
\end{equation}
$$

其中 ELBO 可写为

$$
\begin{equation}
\label{eq:elbo}
\begin{aligned}
L(\theta, \phi; x) &\triangleq \int_z \ln{\left(\frac{p_\theta(x, z)}{q_\phi(z | x)}\right)} q_\phi(z | x) dz \\
&= \int_z \ln{\left(\frac{p_\theta(z)}{q_\phi(z | x)} \cdot p_\theta(x | z) \right)} q_\phi(z | x) dz \\
&= -D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z)) + E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right],
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:elbo2}
L(\theta, \phi; x) = -D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z)) + E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right].
\end{equation}
$$

上式等号右边前一项 KL 散度在 VAE 将编码器和解码器限制为协方差矩阵为对角阵的高斯分布的前提下具有闭合形式的解, 可以实现精确计算; 后一项则需要使用蒙特卡洛方法进行近似, 即

$$
\begin{equation}
\label{eq:monte-carlo}
E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right] \approx \frac{1}{L} \sum_{l = 1}^L \ln{p_\theta(x | z^{(l)})},
\end{equation}
$$

其中 $z^{(l)} \sim q_\phi(z | x), \dbsp l = 1, 2, \dots, L$, $L$ 为采样次数.

由于 \eqref{eq:monte-carlo} 式右边对 $z$ 的采样依赖的是编码器分布 $q_\phi(z | x)$, 因此还需要将 $z$ 的梯度进一步传递到编码器分布的参数 $\phi$ 上. VAE 论文使用所谓的**重参数化**方法将随机变量 $z$ 看作是另一个新的 (与编码器分布的参数 $\phi$ 无关的) 随机变量 $\epsilon$ 的函数 (即令 $z \triangleq g_{\phi, x}(\epsilon)$, 其中 $g_{\phi, x}$ 是某个精心选择的函数), 同时将蒙特卡洛方法由对 $z$ 进行采样改为对 $\epsilon$ 进行采样. 改变后的蒙特卡罗方法的结果依然不变, 而梯度则能够正常传递到 $\phi$ 上, 从而确保了模型训练的正确性.

VAE 论文约定潜在变量 $z$ 的先验分布 $p_\theta(z)$ 为标准高斯分布,

$$
\begin{equation}
p_\theta(z) \triangleq \mathcal{N}(\boldsymbol{0}, I),
\end{equation}
$$

并约定解码器分布 $p_\theta(x | z)$ 为高斯分布. 对于编码器的真实分布 $p_\theta(z | x)$, VAE 论文认为它是难解的 (intractable), 并且假设它具有接近于协方差矩阵为对角阵的高斯分布的形式, 因此约定编码器的近似分布 $q_\phi(z | x)$ 就为协方差矩阵为对角阵的高斯分布, 即

$$
\begin{equation}
q_\phi(z | x) \triangleq \mathcal{N}(z; \mu_\phi(x), \sigma_\phi^2(x)I).
\end{equation}
$$

回到 \eqref{eq:elbo2} 式, 由于已经约定 $q_\phi(z | x)$ 与 $p_\theta(z)$ 均具有高斯分布的形式, KL 散度项可重写为

$$
\begin{equation}
\begin{aligned}
D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z)) &= \int \ln{\left( \frac{q_\phi(z | x)}{p_\theta(z)} \right)} q_\phi(z | x) dz \\
&= \int \ln{\left( \frac{1}{|\sigma_\phi^2I|^{\frac{1}{2}}} e^{-\frac{1}{2} \left( (z - \mu_\phi)^T (\sigma_\phi^2I)^{-1} (z - \mu_\phi) - z^Tz \right)} \right)} q_\phi(z | x) dz \\
&= \int \bigg[ -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \\
&\qquad \qquad \frac{1}{2} [ (z - \mu_\phi)^T (\sigma_\phi^2I)^{-1} (z - \mu_\phi) - z^Tz ] \bigg] q_\phi(z | x) dz \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \int \sum_{d = 1}^D \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \\
&\qquad \qquad \prod_{d = 1}^D \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_1 dz_2 \dots dz_D \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \int \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \\
&\qquad \qquad \prod_{d = 1}^D \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_1 dz_2 \dots dz_D \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \int \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \\
&\qquad \qquad \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_d \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \left[ 1 - \int z_d^2 \cdot \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_d \right] \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \left( 1 - E[z_d^2] \right) \\
&= -\frac{1}{2} \sum_{d = 1}^D \ln{\sigma_{\phi, d}^2} - \frac{1}{2} \sum_{d = 1}^D \left( 1 - (\sigma_{\phi, d}^2 + \mu_{\phi, d}^2) \right) \\
&= -\frac{1}{2} \sum_{d = 1}^D(\ln{\sigma_{\phi, d}^2} + 1 - \sigma_{\phi, d}^2 - \mu_{\phi, d}^2),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
D_{\text{KL}}(q_\phi(z | x) \Vert p_\theta(z)) = -\frac{1}{2} \sum_{d = 1}^D(\ln{\sigma_{\phi, d}^2} + 1 - \sigma_{\phi, d}^2 - \mu_{\phi, d}^2).
\end{equation}
$$

可以看到重写后的 KL 散度项已经能够由编码器分布的期望 $\mu_\phi$ 和方差 $\sigma_\phi^2$ 显式计算得到.

对于 \eqref{eq:monte-carlo} 式的蒙特卡洛项, 由于 $z$ 所服从的分布 $q_\phi(z | x)$ 为协方差矩阵为对角阵的高斯分布, 很自然地可以令函数 $g_{\phi, x}$ 具有平凡的形式,

$$
\begin{equation}
z \triangleq g_{\phi}(\epsilon) = \sigma_\phi(x) \epsilon + \mu_\phi(x).
\end{equation}
$$

于是 $\epsilon \sim \mathcal{N}(0, I)$, 随机变量 $\ln{p_\theta(x | z)}$ 也可被重新看作是随机变量 $\epsilon$ 的函数, 即 $\ln{p_\theta(x | g_{\phi, x}(\epsilon))}$, 进而有

$$
\begin{equation}
\begin{aligned}
E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right] &= E_{\epsilon \sim \mathcal{N}(0, 1)}\left[\ln{p_\theta(x | g_{\phi, x}(\epsilon))}\right] \\
&\approx \frac{1}{L} \sum_{l = 1}^L \ln{p_\theta(x | g_{\phi, x}(\epsilon^{(l)}))},
\end{aligned}
\end{equation}
$$

其中 $\epsilon^{(l)} \sim \mathcal{N}(0, 1), \dbsp l = 1, 2, \dots, L$, $L$ 为采样次数.

## 总结

- [这篇博客](https://blog.csdn.net/a312863063/article/details/87953517)中提到 VAE 的理论基础是高斯混合模型 (GMM), 即 $p(x) = \int p(x | z)p(z) dz$ 可看作是对一系列高斯分布 $p(x | z)$ 关于同样为高斯分布的 $p(z)$ 做积分. 在离散的情况下 $p(x)$ 就是若干个高斯分布的**加权叠加**.
- 训练阶段的 Pipeline:
  1. 潜在变量的先验分布 $p_{\theta}(z)$ 约定为标准多元高斯分布 $\mathcal{N}(z; 0, I)$,解码器分布 $p_{\theta}(x | z)$ 约定为多元高斯分布 (协方差矩阵是否为对角阵原论文里并未提及),编码器分布 $q_{\phi}(z | x)$ 约定为协方差矩阵为对角阵的多元高斯分布.
  1. 为编码器 $q_{\phi}(z | x)$ 和解码器 $p_{\theta}(x | z)$ 选择合适的神经网络结构作为骨架.
  1. 每轮迭代从训练集中抽取 $M$ 张图片构成的 mini-batch. 对于第 $m$ 张图片 $x^{(m)}$, 首先计算 KL 散度, 然后从编码器中采样得到 $L$ 个 $z$ 计算重建损失, 并将 KL 散度和重建损失相加得到第 $m$ 张图片的 ELBO 即损失函数值, 最后通过随机梯度下降法进行训练. 实验表明只要每轮随机梯度下降小批量数据集的大小足够, 即使 $L$ 设置得再小也无妨. 一般将 $L$ 设置为 1, 但需要确保 $M$ 不会太小.

## 参考资料

- [[1312.6114] Auto-Encoding Variational Bayes](https://arxiv.org/abs/1312.6114)
- [Variational Bayesian methods - Wikipedia](https://en.wikipedia.org/wiki/Variational_Bayesian_methods)
- [Bayesian inference - Wikipedia](https://en.wikipedia.org/wiki/Bayesian_inference)
- [Evidence lower bound - Wikipedia](https://en.wikipedia.org/wiki/Evidence_lower_bound)
- [Jensen's inequality - Wikipedia](https://en.wikipedia.org/wiki/Jensen%27s_inequality)
- [Convex function - Wikipedia](https://en.wikipedia.org/wiki/Convex_function)
- [Kullback–Leibler divergence - Wikipedia](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence)
- [Why KL divergence is non-negative? - Stack Exchange](https://stats.stackexchange.com/questions/335197/why-kl-divergence-is-non-negative)
- [Gibbs' inequality - Wikipedia](https://en.wikipedia.org/wiki/Gibbs'_inequality)
- [Monte Carlo method - Wikipedia](https://en.wikipedia.org/wiki/Monte_Carlo_method)
- [Law of large numbers - Wikipedia](https://en.wikipedia.org/wiki/Law_of_large_numbers#Strong_law)

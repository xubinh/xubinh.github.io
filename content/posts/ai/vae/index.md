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

接下来的小节里首先会讨论基础的 KL 散度和蒙特卡洛方法等预备知识, 然后深入讲解变分推断的脉络与背后的缘由, 最后再在这些知识的基础上合拢得到 VAE.

## KL 散度

KL 散度, 全称为 **KullbacKLeibler 散度** (KullbacKLeibler Divergence), 是一种衡量两个概率分布之间差异的**非对称**度量, 一般用于在使用某个分布 $Q$ 逼近某个真实分布 $P$ 时需要对这两个分布之间的差距进行估计的场合下. 在探讨 KL 散度之前首先介绍一下什么是信息熵和交叉熵 (KL 散度也被称为 "相对熵").

### 信息熵

在信息论中, 人们使用随机变量的**信息熵** (entropy) 来衡量该随机变量的分布函数在平均情况下的不确定性程度 (即平均蕴含的信息量大小). 离散情况下, 随机变量 $X$ 的分布函数 $P$ 的信息熵定义为

$$
\newcommand\tr[1]{\mathrm{tr}\left(#1\right)}
\newcommand\dbsp{\ \ }
\newcommand\argmax[1]{\underset{#1}{\operatorname{arg max}}\ }
\newcommand\argmin[1]{\underset{#1}{\operatorname{arg min}}\ }
\newcommand\xbquad{\:\:\:\:\:\:}
\newcommand\xbqquad{\qquad \qquad}
\newcommand\xbb[1]{\boldsymbol{#1}}
\begin{equation}
\label{eq:entropy}
\operatorname{H}(P) \triangleq -\sum_x P(x) \log P(x),
\end{equation}
$$

其中 $x$ 为随机变量 $X$ 的可能结果. 要理解这个定义, 首先需要理解什么是 "随机变量蕴含的信息". 信息论的核心思想是所传达信息的价值取决于该信息内容能够 "令人惊讶" 的程度, 即认为如果一个事件极有可能发生, 该信息包含的价值就很小, 反之如果知道一个事件极不可能发生, 那么该信息包含的信息量就要大得多. 例如知道 "某个特定数字不会是彩票的中奖号码" 所提供的信息量非常小, 因为是个人都知道彩票的中奖概率有多低, 随便选一个数几乎肯定不会中奖, 但反过来知道 "某个特定数字会中奖" 就具有很高的信息量了.

定义一个事件 $\omega$ 蕴含的**信息内容** (information content) (或**自信息** (self-information)) 为

$$
\begin{equation}
I(P(\omega)) \triangleq -\log{P(\omega)},
\end{equation}
$$

其中 $I$ 称为信息函数 (information function). 根据上述定义, $\omega$​ 的信息内容大小与其发生的概率成反比.

信息函数 $I$ 需要满足以下性质:

1. $I$ 单调递减: 事件的概率越大, 包含的信息越少.
1. $I(1) = 0$: 必然事件不包含任何信息.
1. $I(P(\omega_1) \cdot P(\omega_2)) = I(P(\omega_1)) + I(P(\omega_2))$: 独立事件之交的信息内容等于独立事件信息内容之和.

可以证明若要满足上述性质, 信息函数 $I$ 的形式必须为对数函数 $-\log_a{u}$, 其中底数 $a > 1$.

由定义 \eqref{eq:entropy}, 随机变量的信息熵可看作是该随机变量的所有可能结果 (对应的事件) 的信息内容的期望,

$$
\begin{equation}
\operatorname H(P) = -\sum_x P(x) \log P(x) = \mathbb{E}_{x \sim P(\cdot)}[-\log{P(x)}] = \mathbb{E}_{x \sim P(\cdot)}[I(P(x))].
\end{equation}
$$

只需要将离散概率分布 $P$ 替换为概率密度函数 $p$ 并将求和替换为积分即可将上述过程推广至连续型随机变量的情况下.

### 交叉熵

交叉熵是关于随机变量的又一个度量. 离散情况下, 两个分布函数 $P$ 和 $Q$ 的交叉熵定义为

$$
\begin{equation}
H(P, Q) \triangleq -\sum_{x} P(x) \log{Q(x)} = \mathbb{E}_{x \sim P(\cdot)}[-\log{Q(x)}].
\end{equation}
$$

实践中常常将交叉熵用于衡量 $P$ 和 $Q$ 这两个分布函数之间的差异 (其中 $P$ 代表真实 (目标) 分布, $Q$ 代表某个用于近似 $P$ 的参数化的分布), 这一点与交叉熵和 KL 散度之间的关系有关.

### KL 散度及其性质

离散情况下, 两个分布函数 $P$ 和 $Q$ 的 KL 散度定义为

$$
\begin{equation}
\begin{aligned}
D_{\text{KL}}(P(x) || Q(x)) &\triangleq \mathbb{E}_{x \sim P(\cdot)}\left[\log\frac{P(x)}{Q(x)}\right] \\
&= \sum_{x} P(x) \log \frac{P(x)}{Q(x)} \\
&= H(P, Q) - H(P),
\end{aligned}
\end{equation}
$$

推广到连续情况下为

$$
\begin{equation}
D_{\text{KL}}(p(x) || q(x)) \triangleq \mathbb{E}_{x \sim p(\cdot)}\left[\log\frac{p(x)}{q(x)}\right],
\end{equation}
$$

其中 $p$, $q$ 为概率密度函数.

KL 散度满足如下性质:

1. 非负性, 即 $D_{\text{KL}}(P \| Q) \geq 0$, 当且仅当 $P \equiv Q$ 时取等号.
1. 非对称性, 即 $D_{\text{KL}}(P \| Q) \not\equiv D_{\text{KL}}(Q \| P)$.
1. 两个 (多元) 高斯分布的 KL 散度存在闭合形式的计算公式.

性质 1 和性质 3 将在下面给出详细证明. 关于对称性, 容易验证一个反例为两个成功概率分别为 $\frac{1}{2}$ 和 $\frac{1}{3}$ 的伯努利试验.

#### 非负性证明

易知 $\forall x > 0$,

$$
\begin{equation}
\ln x \leq x - 1,
\end{equation}
$$

当且仅当 $x = 1$ 时取等号. 于是离散情况下有

$$
\begin{equation}
\begin{aligned}
D_{\text{KL}}(P(x) || Q(x)) &= \sum_{x} P(x) \log{\frac{P(x)}{Q(x)}} \\
&= -\sum_{x} P(x) \log{\frac{Q(x)}{P(x)}} \\
&\geq -\sum_{x} P(x) \cdot \left(\frac{Q(x)}{P(x)} - 1\right) \\
&= -\sum_{x} P(x) \frac{Q(x)}{P(x)} + \sum_{x} P(x) \\
&= -1 + 1 \\
&= 0,
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
D_{\text{KL}}(P(x) || Q(x)) \geq 0,
\end{equation}
$$

当且仅当 $P$ 与 $Q$ 具有相同分布时取等号. 连续情况同理可证.

由于两个概率分布 $P$ 和 $Q$ 的 KL 散度降至为零当且仅当这两个分布相同, 将 KL 散度作为两个概率分布之间差异的度量是十分自然的.

#### 与交叉熵之间的关系

KL 散度与交叉熵之间关系密切. 实际上两个概率分布 $P$ 和 $Q$ 之间的交叉熵恰由 $P$ 的信息熵与 $P$, $Q$ 之间的 KL 散度构成,

$$
\begin{equation}
\begin{aligned}
H(P, Q) &= \mathbb{E}_{x \sim P(\cdot)}[-\log{Q(x)}] \\
&= \mathbb{E}_{x \sim P(\cdot)}[-\log{P(x)}] + \mathbb{E}_{x \sim P(\cdot)}\left[\frac{\log{P(x)}}{\log{Q(x)}}\right] \\
&= H(P) + D_{\text{KL}}(P || Q),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:kl-cross}
H(P, Q) = H(P) + D_{\text{KL}}(P || Q).
\end{equation}
$$

由 KL 散度的非负性,

$$
\begin{equation}
H(P, Q) - H(P) = D_{\text{KL}}(P || Q) \geq 0,
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:cross-geq-entropy}
H(P) \leq H(P, Q).
\end{equation}
$$

由 \eqref{eq:cross-geq-entropy} 式可知 $P$ 与 $Q$ 之间的交叉熵总是大于 $P$ 的信息熵, 并且差值正是 $P$ 与 $Q$ 之间的 KL 散度.

#### 高斯分布之间的 KL 散度

前面提到两个 (多元) 高斯分布的 KL 散度存在闭合形式的计算公式. 此性质的一个特例将会在推导 VAE 中的 ELBO 时发挥作用, 这里索性直接给出完整推导. 已知服从多元高斯分布 $\mathcal{N}(\mu, \Sigma)$ 的随机向量 $\boldsymbol{X}$ 的概率密度函数为

$$
\begin{equation}
\frac{1}{\sqrt{(2\pi)^k|\Sigma|}} \exp{\left( -\frac{1}{2}(\xbb{x} - \xbb{\mu}^T) \xbb{\Sigma}^{-1} (\xbb{x} - \xbb{\mu}) \right)},
\end{equation}
$$

其中 $k$ 为维数. 于是任意两个多元高斯分布 $\mathcal{N}(\mu_1, \Sigma_1)$ 和 $\mathcal{N}(\mu_2, \Sigma_2)$ 之间的 KL 散度

$$
\begin{equation}
\begin{aligned}
&\xbquad D_{KL}(\mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1) || \mathcal{N}(\xbb{\mu}_2, \xbb{\Sigma}_2)) \\
&= E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \ln{ \left( \frac{\frac{1}{\sqrt{(2\pi)^D|\xbb{\Sigma}_1|}} \exp{\left( -\frac{1}{2}(\xbb{x} - \xbb{\mu}_1^T) \xbb{\Sigma}_1^{-1} (\xbb{x} - \xbb{\mu}_1) \right)}}{\frac{1}{\sqrt{(2\pi)^D|\xbb{\Sigma}_2|}} \exp{\left( -\frac{1}{2}(\xbb{x} - \xbb{\mu}_2^T) \xbb{\Sigma}_2^{-1} (\xbb{x} - \xbb{\mu}_2) \right)}} \right) } \right] \\
&= E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\Bigg[ \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} (\xbb{x} - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} (\xbb{x} - \xbb{\mu}_1) + \\
&\xbqquad \frac{1}{2} (\xbb{x} - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} (\xbb{x} - \xbb{\mu}_2) \Bigg] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} (\xbb{x} - \xbb{\mu}_1) \right] + \\
&\xbqquad \frac{1}{2} E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} (\xbb{x} - \xbb{\mu}_2) \right] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \tr{(\xbb{x} - \xbb{\mu}_1) (\xbb{x} - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1}} \right] + \\
&\xbqquad \frac{1}{2} E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \tr{(\xbb{x} - \xbb{\mu}_2) (\xbb{x} - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1}} \right] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_1) (\xbb{x} - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} \right]} + \\
&\xbqquad \frac{1}{2} \tr{E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_2) (\xbb{x} - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} \right]} \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_1) (\xbb{x} - \xbb{\mu}_1)^T \right] \xbb{\Sigma}_1^{-1}} + \\
&\xbqquad \frac{1}{2} \tr{E_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\xbb{x} - \xbb{\mu}_2) (\xbb{x} - \xbb{\mu}_2)^T \right] \xbb{\Sigma}_2^{-1}} \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{ \xbb{\Sigma}_1 \xbb{\Sigma}_1^{-1}} + \frac{1}{2} \tr{(\xbb{\Sigma}_1 + (\xbb{\mu}_1 - \xbb{\mu}_2)(\xbb{\mu}_1 - \xbb{\mu}_2)^T) \xbb{\Sigma}_2^{-1}} \\
&= \frac{1}{2} \left( \ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - k + \tr{\xbb{\Sigma}_2^{-1} \xbb{\Sigma}_1} + (\xbb{\mu}_2 - \xbb{\mu}_1)^T \xbb{\Sigma}_2^{-1} (\xbb{\mu}_2 - \xbb{\mu}_1) \right),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\begin{aligned}
D_{KL}(\mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1) || &\mathcal{N}(\xbb{\mu}_2, \xbb{\Sigma}_2)) = \\
\frac{1}{2} &\left( \ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - k + \tr{\xbb{\Sigma}_2^{-1} \xbb{\Sigma}_1} + (\xbb{\mu}_2 - \xbb{\mu}_1)^T \xbb{\Sigma}_2^{-1} (\xbb{\mu}_2 - \xbb{\mu}_1) \right).
\end{aligned}
\end{equation}
$$

## 蒙特卡洛方法

**蒙特卡洛方法** (Monte Carlo methods) 是一类通过随机采样得到的结果对数学期望进行估计的方法, 其思想是通过重复多次随机抽样获取随机变量的多个结果, 并使用这些结果的均值作为该随机变量的真实数学期望的近似值.

> - 蒙特卡洛方法的名称 "Monte Carlo" 来源于摩纳哥的一家名为 "Monte Carlo Casino" 的赌场, 该方法的创始者——物理学家 Stanislaw Ulam 借鉴了他的叔叔在这家赌场里赌博时的习惯发明了蒙特卡洛方法.

蒙特卡洛方法的理论保证是**柯尔莫哥洛夫强大数定律** (Kolmogorov's law), 该定律的内容是对于独立同分布的随机变量序列 $\{X_i\}_{i = 1}^{\infty}$,

$$
\begin{equation}
\overline{X}_n \stackrel{a.s.}\longrightarrow E[X_1],
\end{equation}
$$

其中 $\overline{X}_n \triangleq \frac{1}{n} \sum_{i = 1}^n X_i$, 记号 $\stackrel{a.s.}\longrightarrow$ 表示 "几乎处处收敛" (可类比于实分析中可测函数的几乎处处收敛). 换句话说假设同时对所有随机变量进行一次采样得到结果序列 $\{x_i\}_{i=1}^\infty$, 柯尔莫哥洛夫定律说明了均值 $\frac{1}{n}\sum_{i = 1}^n x_i$ 趋向于 $E[X_1]$ 的概率为 $1$ (注意这里每个 $\overline{X}_n$ 的样本空间均为所有 $X_n$ 的样本空间的直和 $\bigoplus_{i = 1}^\infty \Omega_i$).

作为一个示例, 下面使用蒙特卡洛方法计算 $\pi$ 值. 考虑向一个边长为 1 的正方形中随机撒点来计算四分之一圆的面积 (下图摘自[维基百科](https://en.wikipedia.org/wiki/Monte_Carlo_method#Overview)):

<figure style="text-align: center;">
  <div style="display: inline-block;">
  <a href="https://en.wikipedia.org/wiki/Monte_Carlo_method#Overview" target="_blank">
    <img src="images/290px-Pi_monte_carlo_all.gif" alt="蒙特卡罗方法用于近似 $\pi$ 值" style="zoom: 100%; border: 2px solid #eef2f5;" />
  </a>
  </div>
  <figcaption>蒙特卡罗方法用于近似 $\pi$ 值</figcaption>
</figure>

由于正方形的面积为 1, 单次撒点构成的伯努利试验的成功概率 (也即数学期望) 恰为四分之一圆的面积, 因此使用蒙特卡洛方法得到的结果最终将会逼近该面积, 进而间接得到 $\pi$ 值.

[TODO] 随机变量的函数的蒙特卡洛方法.

## 变分推断与 ELBO

变分推断 (Variational Inference) 这一名称的来源是变分法 (calculus of variations). 变分法是数学分析下的一个子领域, 这一领域主要关注如何对泛函进行优化的问题. 泛函 (functional) 指的是将函数映射至实数的函数, 即 "函数的函数", 其形式通常为函数 (及其导数) 的定积分. 变分法的关键思想是在函数上施加**微小变化** (即 "变分") 并观察泛函的相应变化来寻找泛函的极值. 以变分法为基础, 变分推断的目标是在由所有概率分布构成的空间中逼近某个形式复杂的真实分布, 此时有两个问题:

1. 如何定义一个合适的概率分布函数族以便进行优化? 即如何对概率分布空间的一个子空间使用某个 (些) 参数 $\theta$ 进行参数化?
1. 如何评估当前已经得到的概率分布与真实分布之间的差异以便指导优化过程?

对于任意一个概率分布 $p(x)$, 我们可将其看作是可观测数据 $x$ 和一个**潜在变量** (latent variable) 的某个联合分布 $p(x, z)$ 的边际分布, 即

$$
\begin{equation}
p(x) = \int p(x, z) dz = \int p(x | z) p(z),
\end{equation}
$$

其中 $p(z)$ 为潜在变量 $z$ 的先验分布, $p(x | z)$ 为 $x$ 关于 $z$ 的似然. 换句话说任意一个可观测数据 $x$ 出现的概率等于一系列似然 $p(x | z)$ 关于先验分布 $p(z)$ 的 "加权平均" (实际上是积分). 对于第一个问题, 变分推断采用的解决方案就是对似然 $p(x | z)$ 同时关于 $z$ 和 $\theta$ 进行参数化, 此时 $p(x | z)$ 变为 $p_\theta(x | z)$, 在 $\theta$ 固定的情况下每个 $z$ 均有对应的 $p_\theta(x | z)$, 联合分布 $p_\theta(x, z) = p_\theta(x | z) p(z)$ 确定, 联合分布确定边际分布 $p_\theta(x)$ 也确定, 于是一个从参数 $\theta$ 到概率分布空间的一个子空间 (概率函数族) $\{p_\theta(x)\}_\theta$ 的映射也随之确立. 注意这里先验分布 $p(z)$ 的形式是什么并不重要, 因为自由度完全可以由 $\theta$ 包揽, 即使 $p(z)$ 固定, 仍然可以通过优化 $\theta$ 对任意目标分布 $p^*(x)$ 进行拟合. 不失一般性可令 $p(z) = \mathcal{N}(0, 1)$.

解决了参数化的问题, 接下来就需要解决第二个问题也就是如何衡量参数化的分布与真实分布之间的差异大小的问题. 首先可以确定的是变分推断的目标为使用参数化分布 $p_\theta(x)$ 逼近真实分布 $p^*(x)$, 或者用 KL 散度的语言来说就是要解决优化问题

$$
\begin{equation}
\label{eq:opt-1}
\min_\theta \ \  D_{\text{KL}}(p^*(x) || p_\theta(x)).
\end{equation}
$$

由 \eqref{eq:kl-cross} 式,

$$
\begin{equation}
\begin{aligned}
D_{\text{KL}}(p^*(x) || p_\theta(x)) &= H(p^*(x), p_\theta(x)) - H(p^*(x)) \\
&= -E_{x \sim p^*(\cdot)}[\ln{p_\theta(x)}] - H(p^*(x)),
\end{aligned}
\end{equation}
$$

由于真实分布 $p^*(x)$ 固定, 上式最后一行最后一项信息熵 $H(p^*(x))$ 同样为固定值, 于是优化问题 \eqref{eq:opt-1} 可转化为

$$
\begin{equation}
\max_\theta \ \ E_{x \sim p^*(\cdot)}[\ln{p_\theta(x)}],
\end{equation}
$$

继而可由蒙特卡洛方法进行近似并转化为

$$
\begin{equation}
\label{eq:loss-monte}
\max_\theta \ \ \frac{1}{N} \sum_{i = 1}^N \ln{p_\theta(x^{(i)})},
\end{equation}
$$

其中使用训练集 $\{x^{(i)}\}_{i = 1}^N$ 作为真实分布 $p^*(x)$ 的采样结果, $N$ 为训练集大小.

为了对 \eqref{eq:loss-monte} 式进行优化, 我们需要分别对每个训练集数据 $x$ 计算对应的对数似然 $\ln{p_\theta(x)}$. 一般情况下 [TODO], 但在变分推断的情况下我们并不这样做, 而是 [TODO]...

## VAE

VAE 想要求解最优化问题

$$
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
&\triangleq L(\theta, \phi; x) + D_{\text{KL}}(q_\phi(z | x) || p_\theta(z | x)),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:likelyhood2}
\ln{p_\theta(x)} \triangleq L(\theta, \phi; x) + D_{\text{KL}}(q_\phi(z | x) || p_\theta(z | x)),
\end{equation}
$$

其中 ELBO 可写为

$$
\begin{equation}
\label{eq:elbo}
\begin{aligned}
L(\theta, \phi; x) &\triangleq \int_z \ln{\left(\frac{p_\theta(x, z)}{q_\phi(z | x)}\right)} q_\phi(z | x) dz \\
&= \int_z \ln{\left(\frac{p_\theta(z)}{q_\phi(z | x)} \cdot p_\theta(x | z) \right)} q_\phi(z | x) dz \\
&= -D_{\text{KL}}(q_\phi(z | x) || p_\theta(z)) + E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right],
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}
\label{eq:elbo2}
L(\theta, \phi; x) = -D_{\text{KL}}(q_\phi(z | x) || p_\theta(z)) + E_{z \sim q_\phi(z | x)}\left[\ln{p_\theta(x | z)}\right].
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
&\xbquad  D_{\text{KL}}(q_\phi(z | x) || p_\theta(z)) \\
&= \int \ln{\left( \frac{q_\phi(z | x)}{p_\theta(z)} \right)} q_\phi(z | x) dz \\
&= \int \ln{\left( \frac{1}{|\sigma_\phi^2I|^{\frac{1}{2}}} e^{-\frac{1}{2} \left( (z - \mu_\phi)^T (\sigma_\phi^2I)^{-1} (z - \mu_\phi) - z^Tz \right)} \right)} q_\phi(z | x) dz \\
&= \int g[ -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} [ (z - \mu_\phi)^T (\sigma_\phi^2I)^{-1} (z - \mu_\phi) - z^Tz ] g] q_\phi(z | x) dz \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \int \sum_{d = 1}^D \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \\
&\xbqquad \prod_{d = 1}^D \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_1 dz_2 \dots dz_D \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \int \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \\
&\xbqquad \prod_{d = 1}^D \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_1 dz_2 \dots dz_D \\
&= -\frac{1}{2} \ln{|\sigma_\phi^2I|} - \frac{1}{2} \sum_{d = 1}^D \int \left[\frac{1}{\sigma_{\phi, d}^2}(z_d - \mu_{\phi, d})^2 - z_d^2 \right] \cdot \frac{1}{\sqrt{2\pi}\sigma_{\phi, d}} e^{-\frac{(z_d - \mu_{\phi, d})^2}{2\sigma_{\phi, d}^2}} dz_d \\
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
D_{\text{KL}}(q_\phi(z | x) || p_\theta(z)) = -\frac{1}{2} \sum_{d = 1}^D(\ln{\sigma_{\phi, d}^2} + 1 - \sigma_{\phi, d}^2 - \mu_{\phi, d}^2).
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

训练阶段的 Pipeline 总结如下:

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
- [mathematical statistics - KL divergence between two multivariate Gaussians - Cross Validated](https://stats.stackexchange.com/questions/60680/kl-divergence-between-two-multivariate-gaussians)
- [Estimation of covariance matrices - Wikipedia](https://en.wikipedia.org/wiki/Estimation_of_covariance_matrices#The_trace_of_a_1_.C3.97_1_matrix)
- [Multivariate normal distribution - Wikipedia](https://en.wikipedia.org/wiki/Multivariate_normal_distribution)
- [probability - Whats the domain of the sample average in Strong Law of Large Numbers? - Mathematics Stack Exchange](https://math.stackexchange.com/questions/1524489/whats-the-domain-of-the-sample-average-in-strong-law-of-large-numbers)
- Bertsekas, Dimitri, and John Tsitsiklis. _Introduction to Probability_. 2nd ed. Athena Scientific, 2008. ISBN: 9781886529236.
- [ELBO — What & Why | Yunfan’s Blog](https://yunfanj.com/blog/2021/01/11/ELBO.html)

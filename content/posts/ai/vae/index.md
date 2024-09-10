---
title: "VAE (变分自编码器) 学习笔记"
description: "浅谈 VAE 背后的设计思想与缘由"
summary: "浅谈 VAE 背后的设计思想与缘由"
date: 2024-08-19T17:49:24+08:00
draft: false
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

KL 散度, 全称为 **Kullback–Leibler 散度** (Kullback–Leibler Divergence), 是一种衡量两个概率分布之间差异的非对称度量, 一般用于在使用某个分布 $Q$ 逼近某个真实分布 $P$ 时需要对这两个分布之间的差距进行估计的场合下. 在探讨 KL 散度之前首先介绍一下什么是信息熵和交叉熵 (KL 散度也被称为 "相对熵").

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
\newcommand\btheta{\xbb{\theta}}
\newcommand\bphi{\xbb{\phi}}
\newcommand\bz{\xbb{z}}
\newcommand\bx{\xbb{x}}
\newcommand\bX{\xbb{X}}
\newcommand\bmu{\xbb{\mu}}
\newcommand\bsigma{\xbb{\sigma}}
\newcommand\bepsilon{\xbb{\epsilon}}
\newcommand\bzero{\xbb{0}}
\newcommand\bI{\xbb{I}}
\newcommand\overbar[1]{\mkern 1.5mu\overline{\mkern-1.5mu#1\mkern-1.5mu}\mkern 1.5mu}
\begin{equation}\label{eq:entropy}
\operatorname{H}(P) \triangleq -\sum_x P(x) \log P(x),
\end{equation}
$$

其中 $x$ 为随机变量 $X$ 的可能结果. 要理解这个定义, 首先需要理解什么是 "随机变量蕴含的信息". 信息论的核心思想是所传达信息的价值取决于该信息内容能够 "令人惊讶" 的程度, 即认为如果一个事件极有可能发生, 该信息包含的价值就很小, 反之如果知道一个事件极不可能发生, 那么该信息包含的信息量就要大得多. 例如知道 "某个特定数字不会是彩票的中奖号码" 所提供的信息量非常小, 因为是个人都知道彩票的中奖概率有多低, 随便选一个数几乎肯定不会中奖, 但反过来知道 "某个特定数字会中奖" 就具有很高的信息量了.

定义一个事件 $\omega$ 蕴含的信息内容 (information content) (或自信息 (self-information)) 为

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

实践中常常将交叉熵用于衡量 $P$ 和 $Q$ 这两个分布函数之间的差异 (其中 $P$ 代表真实 (目标) 分布, $Q$ 代表某个用于近似 $P$ 的参数化的分布), 这一点与交叉熵和 KL 散度之间的关系有关. 交叉熵同样可推广至连续型随机变量的情况下.

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
\begin{equation}\label{eq:kl-cross}
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
\begin{equation}\label{eq:cross-geq-entropy}
H(P) \leq H(P, Q).
\end{equation}
$$

由 \eqref{eq:cross-geq-entropy} 式可知 $P$ 与 $Q$ 之间的交叉熵总是大于 $P$ 的信息熵, 并且差值正是 $P$ 与 $Q$ 之间的 KL 散度.

#### 高斯分布之间的 KL 散度

前面提到两个 (多元) 高斯分布的 KL 散度存在闭合形式的计算公式. 此性质的一个特例将会在推导 VAE 中的 ELBO 时发挥作用, 这里索性直接给出完整推导. 已知服从多元高斯分布 $\mathcal{N}(\mu, \Sigma)$ 的随机向量 $\boldsymbol{X}$ 的概率密度函数为

$$
\begin{equation}
\frac{1}{\sqrt{(2\pi)^k|\Sigma|}} \exp{\left( -\frac{1}{2}(\bx - \xbb{\mu}^T) \xbb{\Sigma}^{-1} (\bx - \xbb{\mu}) \right)},
\end{equation}
$$

其中 $k$ 为维数. 于是任意两个多元高斯分布 $\mathcal{N}(\mu_1, \Sigma_1)$ 和 $\mathcal{N}(\mu_2, \Sigma_2)$ 之间的 KL 散度

$$
\begin{equation}
\begin{aligned}
&\xbquad D_{KL}(\mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1) || \mathcal{N}(\xbb{\mu}_2, \xbb{\Sigma}_2)) \\
&= \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \ln{ \left( \frac{\frac{1}{\sqrt{(2\pi)^D|\xbb{\Sigma}_1|}} \exp{\left( -\frac{1}{2}(\bx - \xbb{\mu}_1^T) \xbb{\Sigma}_1^{-1} (\bx - \xbb{\mu}_1) \right)}}{\frac{1}{\sqrt{(2\pi)^D|\xbb{\Sigma}_2|}} \exp{\left( -\frac{1}{2}(\bx - \xbb{\mu}_2^T) \xbb{\Sigma}_2^{-1} (\bx - \xbb{\mu}_2) \right)}} \right) } \right] \\
&= \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\Bigg[ \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} (\bx - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} (\bx - \xbb{\mu}_1) \\
&\xbqquad + \frac{1}{2} (\bx - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} (\bx - \xbb{\mu}_2) \Bigg] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} (\bx - \xbb{\mu}_1) \right] \\
&\xbqquad + \frac{1}{2} \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} (\bx - \xbb{\mu}_2) \right] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \tr{(\bx - \xbb{\mu}_1) (\bx - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1}} \right] \\
&\xbqquad + \frac{1}{2} \mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ \tr{(\bx - \xbb{\mu}_2) (\bx - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1}} \right] \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{\mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_1) (\bx - \xbb{\mu}_1)^T \xbb{\Sigma}_1^{-1} \right]} \\
&\xbqquad + \frac{1}{2} \tr{\mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_2) (\bx - \xbb{\mu}_2)^T \xbb{\Sigma}_2^{-1} \right]} \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{\mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_1) (\bx - \xbb{\mu}_1)^T \right] \xbb{\Sigma}_1^{-1}} \\
&\xbqquad + \frac{1}{2} \tr{\mathbb{E}_{x \sim \mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1)}\left[ (\bx - \xbb{\mu}_2) (\bx - \xbb{\mu}_2)^T \right] \xbb{\Sigma}_2^{-1}} \\
&= \frac{1}{2}\ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - \frac{1}{2} \tr{ \xbb{\Sigma}_1 \xbb{\Sigma}_1^{-1}} + \frac{1}{2} \tr{(\xbb{\Sigma}_1 + (\xbb{\mu}_1 - \xbb{\mu}_2)(\xbb{\mu}_1 - \xbb{\mu}_2)^T) \xbb{\Sigma}_2^{-1}} \\
&= \frac{1}{2} \left( \ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - k + \tr{\xbb{\Sigma}_2^{-1} \xbb{\Sigma}_1} + (\xbb{\mu}_2 - \xbb{\mu}_1)^T \xbb{\Sigma}_2^{-1} (\xbb{\mu}_2 - \xbb{\mu}_1) \right),
\end{aligned}
\end{equation}
$$

即

$$
\begin{equation}\label{eq:closed-form-kl}
\begin{aligned}
D_{KL}(\mathcal{N}(\xbb{\mu}_1, \xbb{\Sigma}_1) || &\mathcal{N}(\xbb{\mu}_2, \xbb{\Sigma}_2)) = \\
\frac{1}{2} &\left( \ln{\frac{|\xbb{\Sigma}_2|}{|\xbb{\Sigma}_1|}} - k + \tr{\xbb{\Sigma}_2^{-1} \xbb{\Sigma}_1} + (\xbb{\mu}_2 - \xbb{\mu}_1)^T \xbb{\Sigma}_2^{-1} (\xbb{\mu}_2 - \xbb{\mu}_1) \right),
\end{aligned}
\end{equation}
$$

其中利用到了两个任意矩阵 $A$ 和 $B$ 的乘积的**迹** (trace) 的性质, $\tr{AB} = \tr{BA}$.

## 蒙特卡洛方法

**蒙特卡洛方法** (Monte Carlo methods) 是一类通过随机采样得到的结果对数学期望进行估计的方法, 其思想是通过重复多次随机抽样获取随机变量的多个结果, 并使用这些结果的均值作为该随机变量的真实数学期望的近似值.

> - 蒙特卡洛方法的名称 "Monte Carlo" 来源于摩纳哥的一家名为 "Monte Carlo Casino" 的赌场, 该方法的创始者——物理学家 Stanislaw Ulam 借鉴了他的叔叔在这家赌场里赌博时的习惯发明了蒙特卡洛方法.

蒙特卡洛方法的理论保证是**柯尔莫哥洛夫强大数定律** (Kolmogorov's law), 该定律的内容是对于独立同分布的随机变量序列 $\{X_i\}_{i = 1}^{\infty}$,

$$
\begin{equation}
\overbar{X}_n \stackrel{a.s.}\longrightarrow E[X_1],
\end{equation}
$$

其中 $\overbar{X}_n \triangleq \frac{1}{n} \sum_{i = 1}^n X_i$, 记号 $\stackrel{a.s.}\longrightarrow$ 表示 "几乎处处收敛" (可类比于实分析中可测函数的几乎处处收敛). 换句话说假设同时对所有随机变量进行一次采样得到结果序列 $\{x_i\}_{i=1}^\infty$, 柯尔莫哥洛夫定律说明了均值 $\frac{1}{n}\sum_{i = 1}^n x_i$ 趋向于 $E[X_1]$ 的概率为 $1$ (注意这里每个 $\overbar{X}_n$ 的样本空间均为所有 $X_n$ 的样本空间的直和 $\bigoplus_{i = 1}^\infty \Omega_i$).

作为一个示例, 下面使用蒙特卡洛方法计算 $\pi$ 值. 考虑向一个边长为 1 的正方形中随机撒点来计算四分之一圆的面积 (下图摘自<a href="https://en.wikipedia.org/wiki/Monte_Carlo_method#Overview" target="_blank">维基百科</a>):

<figure style="text-align: center;">
  <div style="display: inline-block;">
  <a href="https://en.wikipedia.org/wiki/Monte_Carlo_method#Overview" target="_blank">
    <img src="images/290px-Pi_monte_carlo_all.gif" alt="蒙特卡罗方法用于近似 $\pi$ 值" style="zoom: 100%; border: 2px solid #eef2f5;" />
  </a>
  </div>
  <figcaption>蒙特卡罗方法用于近似 $\pi$ 值</figcaption>
</figure>

由于正方形的面积为 1, 单次撒点构成的伯努利试验的成功概率 (也即数学期望) 恰为四分之一圆的面积, 因此使用蒙特卡洛方法得到的结果最终将会逼近该面积, 进而间接得到 $\pi$ 值.

## 变分推断与 ELBO

**变分推断** (Variational Inference) 这一名称的来源是变分法 (calculus of variations). 变分法是数学分析下的一个子领域, 这一领域主要关注如何对泛函进行优化的问题. 泛函 (functional) 指的是将函数映射至实数的函数, 即 "函数的函数", 其形式通常为函数 (及其导数) 的定积分. 变分法的关键思想是在函数上施加微小变化 (即 "变分") 并观察泛函的相应变化来寻找泛函的极值. 以变分法为基础, 变分推断的目标是在由所有概率分布构成的空间中逼近某个形式复杂的真实分布, 此时有两个问题:

1. 如何定义一个合适的概率分布函数族以便进行优化? 即如何对概率分布空间的一个子空间使用某些参数 $\theta$ 进行参数化?
1. 如何评估当前已经得到的概率分布与真实分布之间的差异以便指导优化过程?

对于任意一个概率分布 $p(x)$, 我们可将其看作是可观测数据 $x$ 和一个**潜在变量** (latent variable) $z$ 的某个联合分布 $p(x, z)$ 的边际分布,

$$
\begin{equation}
p(x) = \int p(x, z) dz = \int p(x | z) p(z) dz,
\end{equation}
$$

其中 $p(z)$ 为 $z$ 的先验分布, $p(x | z)$ 为 $x$ 关于 $z$ 的似然. 换句话说任意一个可观测数据 $x$ 出现的概率 $p(x)$ 等于一系列似然 $p(x | z)$ 关于服从先验分布 $p(z)$ 的 $z$ 的 "加权平均" (实际上是积分). 对于第一个问题, 变分推断采用的解决方案是对似然 $p(x | z)$ 同时关于 $z$ 和 $\theta$ 进行参数化, 此时 $p(x | z)$ 变为 $p_\theta(x | z)$, 在 $\theta$ 固定的情况下, 只要给定一个 $z$, 就能确定对应的分布 $p_\theta(x | z)$, 所以联合分布 $p_\theta(x, z) = p_\theta(x | z) p(z)$ 就确定下来了, 联合分布确定边际分布 $p_\theta(x)$ 也确定, 于是一个从参数 $\theta$ 到概率分布空间的一个子空间 (概率函数族) $\{p_\theta(x)\}_\theta$ 的映射也确定下来. 这里先验分布 $p(z)$ 的形式是什么并不重要, 因为自由度完全可以由 $\theta$ 包揽, 无论 $p(z)$ 是什么形式均可以通过优化 $\theta$ 来 "抹平" 差异, 并对目标分布 $p^*(x)$ 进行拟合. 不失一般性可令 $p(z) = \mathcal{N}(0, 1)$.

解决了参数化的问题, 接下来就是如何对参数化分布与真实分布之间的差异进行衡量的问题. 首先变分推断的根本目标是用参数化分布 $p_\theta$ 逼近真实分布 $p^*$, 用 KL 散度的语言来讲就是要解决优化问题

$$
\begin{equation}\label{eq:opt-1}
\min_\theta \ \  D_{\text{KL}}(p^*(x) || p_\theta(x)).
\end{equation}
$$

由 \eqref{eq:kl-cross} 式,

$$
\begin{equation}
\begin{aligned}
D_{\text{KL}}(p^*(x) || p_\theta(x)) &= H(p^*(x), p_\theta(x)) - H(p^*(x)) \\
&= -\mathbb{E}_{x \sim p^*(\cdot)}[\ln{p_\theta(x)}] - H(p^*(x)),
\end{aligned}
\end{equation}
$$

由于真实分布 $p^*(x)$ 固定, 上式最后一行最后一项信息熵 $H(p^*(x))$ 同样为固定值, 于是优化问题 \eqref{eq:opt-1} 可转化为

$$
\begin{equation}
\max_\theta \ \ \mathbb{E}_{x \sim p^*(\cdot)}[\ln{p_\theta(x)}],
\end{equation}
$$

进而可通过蒙特卡洛方法进行近似并转化为

$$
\begin{equation}\label{eq:loss-monte}
\max_\theta \ \ \frac{1}{N} \sum_{i = 1}^N \ln{p_\theta(x^{(i)})},
\end{equation}
$$

其中将训练集 $\{x^{(i)}\}_{i = 1}^N$ 作为真实分布 $p^*(x)$ 的采样结果, $N$ 为训练集大小.

为了对 \eqref{eq:loss-monte} 式进行优化, 我们需要分别对每个训练集数据 $x^{(i)}$ 计算对应的对数 (边际) 似然 $\ln{p_\theta(x^{(i)})}$. 一个自然的想法是通过蒙特卡洛方法对 $x$ 的似然关于 $z$ 的先验进行采样,

$$
\begin{equation}
\begin{aligned}
p_\theta(x) &= \int p_\theta(x, z) dz \\
&= \int p_\theta(x | z) p(z) dz \\
&= \mathbb{E}_{z \sim p(\cdot)}[p_\theta(x | z)] \\
&\approx \frac{1}{K} \sum_{i = 1}^K p_\theta(x | z^{(i)}), \xbquad z^{(i)} \sim p(\cdot).
\end{aligned}
\end{equation}
$$

但这样做有些问题, 因为 $p(z)$ 的定义域是整个实数轴, 而 $p_\theta(x | z)$ 的值有可能只在实数轴上很小的一个范围内有比较明显的起伏并且其他地方均接近于零, 在资源有限因而 $K$ 有限的情况下上述采样很难说能有多好的效果. 更好的方法是使用<a href="https://en.wikipedia.org/wiki/Importance_sampling" target="_blank">重要性采样</a>, 即根据某个特殊的关于 $x$ 的后验 $q(z | x)$ 进行采样,

$$
\begin{equation}
\begin{aligned}
p_\theta(x) &= \int p_\theta(x | z) p(z) dz \\
&= \int p_\theta(x | z) \frac{p(z)}{q(z | x)} q(z | x) dz \\
&= \mathbb{E}_{z \sim q(\cdot | x)}\left[p_\theta(x | z) \frac{p(z)}{q(z | x)}\right] \\
&\approx \frac{1}{K} \sum_{i = 1}^K p_\theta(x | z^{(i)}) \frac{p(z^{(i)})}{q(z^{(i)} | x)}, \xbquad z^{(i)} \sim q(\cdot | x).
\end{aligned}
\end{equation}
$$

显然最好的 $q(z | x)$ 就是 $p_\theta(z | x)$, 此时

$$
\begin{equation}
\frac{1}{K} \sum_{i = 1}^K p_\theta(x | z^{(i)}) \frac{p(z^{(i)})}{p_\theta(z^{(i)} | x)} = \frac{1}{K} \sum_{i = 1}^K p_\theta(x) \equiv p_\theta(x), \xbquad z^{(i)} \sim q(\cdot | x).
\end{equation}
$$

问题是 $p_\theta(z | x)$ 和 $p_\theta(x)$ 一样都是未知的, 都需要通过 $p_\theta(x | z)$ 和 $p(z)$ 间接求得, 并且求 $p_\theta(z | x)$ 反倒还需要用到 $p_\theta(x)$, 这就陷入了死循环. 为了解决这一问题, 变分推断引入由另一组参数 $\phi$ 参数化的概率函数族 $q_\phi(z | x)$ 对真实后验 $p_\theta(z | x)$ 进行近似. 这样对 $p_\theta(x)$ 的估计的问题就解决了, 但现在又多了一个优化 $\phi$ 以便逼近 $p_\theta(z | x)$ 的问题, 不过这个问题解决起来很简单, **再加一个关于 $\phi$ 的损失函数就行了**. 注意到

$$
\begin{equation}\label{eq:elbo-1}
\begin{aligned}
\ln{p_\theta(x)} &= \ln{\mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \frac{p_\theta(x, z)}{q_\phi(z | x)} \right]} \\
&\geq \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{q_\phi(z | x)}} \right],
\end{aligned}
\end{equation}
$$

其中不等号由<a href="https://en.wikipedia.org/wiki/Jensen%27s_inequality" target="_blank"><b>詹森不等式</b></a> (Jensen's inequality) 得到, 并且实际上二者的差

$$
\begin{equation}\label{eq:elbo-2}
\begin{aligned}
&\xbquad \ln{p_\theta(x)} - \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{q_\phi(z | x)}} \right] \\
&= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{p_\theta(z | x)}} \right] - \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{q_\phi(z | x)}} \right] \\
&= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{q_\phi(z | x)}{p_\theta(z | x)}} \right]\\
&= D_{\text{KL}}(q_\phi(z | x) || p_\theta(z | x)),
\end{aligned}
\end{equation}
$$

如果定义

$$
\begin{equation}\label{eq:elbo}
\mathcal{L}(\theta, \phi; x) \triangleq \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{q_\phi(z | x)}} \right],
\end{equation}
$$

那么由 \eqref{eq:elbo-1} 式和 \eqref{eq:elbo-2} 式可知 $\ln{p_\theta(x)}$ 总是不会低于 $\mathcal{L}(\theta, \phi; x)$ (即 \eqref{eq:elbo} 式定义了 $\ln{p_\theta(x)}$ 的一个下界 (lower bound)), 并且它们的差恰好就是近似后验 $q_\phi(z | x)$ 与真实后验 $p_\theta(z | x)$ 之间的 KL 散度, $D_{\text{KL}}(q_\phi(z | x) || p_\theta(z | x))$. 由此可知对 $\mathcal{L}(\theta, \phi; x)$ 进行最大化的效果等价于在最大化 $\ln{p_\theta(x)}$ 的同时不断令近似后验 $q_\phi(z | x)$ 逼近真实后验 $p_\theta(z | x)$. 于是只需要将 $\mathcal{L}(\theta, \phi; x)$ 作为损失函数即可同时对 $\theta$ 与 $\phi$ 进行优化. 由于 $\ln{p_\theta(x)}$ 被称作 $x$ 的证据 (evidence), $\mathcal{L}(\theta, \phi; x)$ 也因此被称作**证据下界** (evidence lower bound, **ELBO**).

到目前为止整个推导基本上是顺理成章的, 不过有一点还是需要额外指出. 注意到原先设置 $q_\phi$ 的目的就是为了求 $\ln{p_\theta(x)}$, 现在 $\ln{p_\theta(x)}$ 却不见了. 这乍看上去好像是捡了芝麻丢了西瓜, 但说到底计算具体的 $\ln{p_\theta(x)}$ 值本身还是为了比较不同的 $\theta$ 之间的优劣, 换成计算 $\ln{p_\theta(x)} + C$ 把 $C$ 代入包括常数在内的任何不影响比较的值也一样, 因此形式是无所谓的. 关键在于计算 ELBO 同时还附带了计算 $D_{\text{KL}}(q_\phi(z | x) || p_\theta(z | x))$, 这是选择 ELBO 的主要原因. 如果存在另一个独立的闭合形式的损失函数能够计算这个 KL 散度, 那完全可以把这个独立的损失函数和 $\ln{p_\theta(x)}$ 相加得到的和作为新的损失函数, 但这又绕回来了——这个和实际上就是 ELBO.

我们还可以从其他一些角度观察 ELBO 的性质. 注意到

$$
\begin{align}
\mathcal{L}(\theta, \phi; x) &= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p_\theta(x, z)}{q_\phi(z | x)}} \right] \notag \\
&= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{p_\theta(x, z)} \right] + \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ -\ln{q_\phi(z | x)} \right] \notag \\
&= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{p_\theta(x, z)} \right] + H(q_\phi(z | x)) \label{eq:elbo-rewrite-1}\\
&= \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{\frac{p(z)}{q_\phi(z | x)}} \right] + \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{p_\theta(x | z)} \right] \notag \\
&= -D_{\text{KL}}(q_\phi(z | x) || p(z)) + \mathbb{E}_{z \sim q_\phi(\cdot | x)} \left[ \ln{p_\theta(x | z)} \right] \label{eq:elbo-rewrite-2},
\end{align}
$$

首先观察 \eqref{eq:elbo-rewrite-1} 式, 该式第一项的数学期望将引导 $q_\phi(z | x)$ 尽可能往 "将所有概率密度集中在一个点 $z^*$ 附近" 的方向上优化, 其中

$$
z^* = \argmax{z}\ \ln{p_\theta(x, z)},
$$

如果 $q_\phi(z | x)$ 的形式是高斯分布, 那么它的波峰将逐渐向 $z^*$ 集中并且最终将向上趋于无限 (其极限恰为<a href="https://en.wikipedia.org/wiki/Dirac_delta_function" target="_blank">狄拉克 $\delta$ 函数</a>). 另一方面, \eqref{eq:elbo-rewrite-1} 式第二项的信息熵则将 $q_\phi(z | x)$ 往另一个方向, 即 "将概率密度尽可能摊平" 的方向进行优化. 仍然假设 $q_\phi(z | x)$ 为高斯分布, 易知其<a href="https://en.wikipedia.org/wiki/Differential_entropy#Differential_entropies_for_various_distributions" target="_blank">信息熵</a>为 $\ln{(\sqrt{2\pi e}\sigma)}$, 信息熵越大, 标准差 $\sigma$ 就越大, 概率密度就越分散. 因此直观上看 \eqref{eq:elbo-rewrite-1} 式希望引导的 $q_\phi(z | x)$ 的形式是概率密度往 $z^*$ 集中但又不至于太集中, 处于某种 "平衡" 之下. 这个 "平衡" 指的就是函数族 $q_\phi(z | x)$ 形成的子空间中到 $p_\theta(z | x)$ 的距离最短的那个点.

再来观察 \eqref{eq:elbo-rewrite-2} 式, 该式第一项 KL 散度将 $q_\phi(z | x)$ 往 $p(z)$ 的形式上拉扯, 这可以看作是对 $q_\phi(z | x)$ 的正则化, 而第二项关于对数似然 $\ln{p_\theta(x | z)}$ 的期望被称作关于 $x$ 的**重建损失** (reconstruction loss), 注意这里是直接对 $\ln{p_\theta(x | z)}$ 求期望, 也就是对边际似然 $\ln{p_\theta(x)}$ 的重建.

至此变分推断和 ELBO 是什么已经大概摸清了, 接下来推导 VAE 就是水到渠成的事了.

## VAE

和变分推断类似, VAE 想要利用已知的训练集 $\{\bx^{(i)}\}_{i = 1}^N$ 在参数化的概率分布族 $p_\btheta$ 中进行优化并逼近真实的数据分布 $p^*$, 等价于求解优化问题 (见 \eqref{eq:loss-monte} 式)

$$
\begin{equation}
\max_\btheta \dbsp \frac{1}{N} \sum_{i = 1}^N \ln{p_{\btheta}(\bx^{(i)})},
\end{equation}
$$

单个样本的对数似然 $\ln{p_\btheta(\bx)}$ 等于 ELBO 加上近似后验 $q_{\bphi}(\bz | \bx)$ 与真实后验 $p_{\btheta}(\bz | \bx)$ 之间的 KL 散度 (见 \eqref{eq:elbo-2} 式)

$$
\begin{equation}
\ln{p_\btheta(\bx)} = \mathcal{L}(\btheta, \bphi; \bx) + D_{\text{KL}}(q_\bphi(\bz | \bx) || p_\btheta(\bz | \bx)),
\end{equation}
$$

其中 ELBO 又等于正则化项加上重建损失 (见 \eqref{eq:elbo-rewrite-2} 式)

$$
\begin{equation}\label{eq:elbo-rewrite-2-mul-dim}
\mathcal{L}(\btheta, \bphi; \bx) = -D_{\text{KL}}(q_\bphi(\bz | \bx) || p_\btheta(\bz)) + \mathbb{E}_{\bz \sim q_\bphi(\cdot | \bx)} \left[ \ln{p_\btheta(\bx | \bz)} \right].
\end{equation}
$$

从编码理论的角度来看, 潜在变量 $\bz$ 可看作是对可观察变量 $\bx$ 的一种编码 (code), 因此 VAE 将后验分布 $q_\bphi(\bz | \bx)$ 称为概率性**编码器** (encoder), 同时将似然 $p_\btheta(\bx | \bz)$ 称为概率性**解码器** (decoder), 其中编码器负责将 $\bx$ 压缩为 $\bz$, 而解码器负责将 $\bz$ 重建为 $\bx$.

VAE 约定潜在变量 $\bz$ 的先验分布 $p_\btheta(\bz)$ 为标准高斯分布,

$$
\begin{equation}
p_\btheta(\bz) \triangleq \mathcal{N}(\bzero, \bI),
\end{equation}
$$

并约定解码器 $p_\btheta(\bx | \bz)$ 的形式为高斯分布. 对于真实后验 $p_\btheta(\bz | \bx)$, VAE 假设其形式接近于协方差矩阵为对角阵的高斯分布, 因此约定编码器 $q_\bphi(\bz | \bx)$ 的形式就为协方差矩阵为对角阵的高斯分布, 即

$$
\begin{equation}
q_\bphi(\bz | \bx) \triangleq \mathcal{N}(\bz; \bmu_\bphi(\bx), \bsigma_\bphi^2(\bx)\bI),
\end{equation}
$$

其中 $\bmu_\bphi(\bx)$ 和 $\bsigma_\bphi^2(\bx)$ (上标表示按元素平方) 为由 $\bphi$ 参数化的关于 $\bx$ 的函数. 由于 $q_\bphi(\bz | \bx)$ 和 $p_\btheta(\bz)$ 均约定为高斯分布的形式, 由 \eqref{eq:closed-form-kl} 式立即可以得到 \eqref{eq:elbo-rewrite-2-mul-dim} 式中的 KL 散度项的闭合形式的解

$$
\begin{equation}\label{eq:regularization}
D_{\text{KL}}(q_\bphi(\bz | \bx) || p_\btheta(\bz)) = -\frac{1}{2} \sum_{d = 1}^D(\ln{\bsigma_{\bphi, d}^2(\bx)} + 1 - \bsigma_{\bphi, d}^2(\bx) - \bmu_{\bphi, d}^2(\bx)),
\end{equation}
$$

其中 $D$ 为向量维数, 下标 $d$ 表示向量的第 $d$ 维分量. 可以看到重写后的 KL 散度项能够直接通过 $q_\bphi(\bz | \bx)$ 的期望 $\bmu_\bphi$ 和方差 $\bsigma_\bphi^2$ 计算得到.

另一方面 \eqref{eq:elbo-rewrite-2-mul-dim} 式中的重建损失需要使用蒙特卡洛方法进行近似,

$$
\begin{equation}\label{eq:monte-carlo}
\mathbb{E}_{\bz \sim q_\bphi(\cdot | \bx)}\left[\ln{p_\btheta(\bx | \bz)}\right] \approx \frac{1}{L} \sum_{l = 1}^L \ln{p_\btheta(\bx | \bz^{(l)})},
\end{equation}
$$

其中 $\bz^{(l)} \sim q_\bphi(\cdot | \bx), \dbsp l = 1, 2, \dots, L$, $L$ 为采样次数. 由于 \eqref{eq:monte-carlo} 式右边对 $\bz$ 的采样依赖的是分布 $q_\bphi(\bz | \bx)$, 这里还得想办法把 $\bz$ 的梯度向下传播到参数 $\bphi$ 上. VAE 使用所谓的**重参数化** (Reparameterization) 方法, 把随机变量 $\bz$ 看作是另一个新的随机变量 $\bepsilon$ 的函数, 其中参数 $\bphi$ 被全部保留到函数中, 而 $\bepsilon$ 本身与参数 $\bphi$ 无关 (即令 $\bz \triangleq g_{\bphi, \bx}(\bepsilon)$, 其中 $g_{\bphi, \bx}$ 是某个精心选择的关于 $\bphi$ 的函数), 这样不仅原先关于某个函数 $f(\bz)$ 的蒙特卡洛方法可以转化为等价的关于函数 $(f(g_{\bphi, \bx}(\bepsilon)))$ 的蒙特卡洛方法, 同时还使得梯度能够成功传播到 $\bphi$ 上. 例如对于一维高斯分布 $z \sim \mathcal{N}(\mu, \sigma^2)$, 令 $\epsilon \sim \mathcal{N}(0, 1)$ 并令 $g(\epsilon) = \sigma\epsilon + \mu$ (暂时不考虑 $\phi$ 和 $x$, 因为对这两个变量的处理只不过是重参数法的副作用), 易知 $z \equiv g(\epsilon)$, 并且随机变量 $\eta \triangleq f(z) = f(g(\epsilon))$ 既是 $z$ 的函数又是 $\epsilon$ 的函数, 于是

$$
\begin{align}\label{eq:reparameterization}
\mathbb{E}_{z \sim \mathcal{N}(\mu, \sigma^2)}[f(z)] &= \mathbb{E}[\eta] &&(\eta = f(z))\\
&=\lim_{n \to \infty} \frac{1}{n} \sum_{i = 1}^n \eta^{(i)} &&(\text{Monte Carlo})\\
&=\lim_{n \to \infty} \frac{1}{n} \sum_{i = 1}^n f(g(\epsilon^{(i)})). &&(\eta = f(g(\epsilon)))
\end{align}
$$

回到 \eqref{eq:monte-carlo} 式, 由于 $\bz$ 所服从的分布 $q_\bphi(\bz | \bx)$ 约定为协方差矩阵为对角阵的高斯分布 $\mathcal{N}(\bmu_{\phi}(\bx), \bsigma_{\bphi}^2(\bx) \bI)$, 很自然地可以令函数 $g_{\bphi, \bx}$ 将 $\bz$ 打回到标准高斯分布,

$$
\begin{equation}
\bz \triangleq g_{\bphi, \bx}(\bepsilon) = \bsigma_\bphi(\bx) \odot \bepsilon + \bmu_\bphi(\bx).
\end{equation}
$$

其中 $\epsilon \sim \mathcal{N}(0, I)$, $\odot$ 表示向量之间的按元素相乘. 于是由 \eqref{eq:reparameterization} 式 (的多元推广),

$$
\begin{equation}\label{eq:reconstruction}
\mathbb{E}_{\bz \sim q_\bphi(\cdot | \bx)}\left[\ln{p_\btheta(\bx | \bz)}\right] \approx \frac{1}{L} \sum_{l = 1}^L \ln{p_\btheta(\bx | g_{\bphi, \bx}(\bepsilon^{(l)}))},
\end{equation}
$$

其中 $\epsilon^{(l)} \sim \mathcal{N}(0, 1), \dbsp l = 1, 2, \dots, L$, $L$ 为采样次数. 可以看到改变后的蒙特卡罗方法的结果依然不变, 但梯度已经能够正常传递到 $\bphi$ 上, 从而确保了模型训练结构的正确性.

结合 \eqref{eq:regularization} 式和 \eqref{eq:reconstruction} 式, 单个训练集样本 $\bx$ 的 ELBO 的最终形式为

$$
\begin{equation}\label{eq:elbo-final}
\begin{aligned}
\mathcal{L}(\btheta, \bphi; \bx) &= -D_{\text{KL}}(q_\bphi(\bz | \bx) || p_\btheta(\bz)) + \mathbb{E}_{\bz \sim q_\bphi(\cdot | \bx)} \left[ \ln{p_\btheta(\bx | \bz)} \right] \\
&\approx \frac{1}{2} \sum_{d = 1}^D(\ln{\bsigma_{\bphi, d}^2(\bx)} + 1 - \bsigma_{\bphi, d}^2(\bx) - \bmu_{\bphi, d}^2(\bx)) \\
&\xbqquad + \frac{1}{L} \sum_{l = 1}^L \ln{p_\btheta(\bx | g_{\bphi, \bx}(\bepsilon^{(l)}))},
\end{aligned}
\end{equation}
$$

小批量样本的损失函数为

$$
\begin{equation}
\mathcal{L}(\btheta, \bphi; \bX_{1:M}) = \frac{1}{M} \sum_{i = 1}^M \mathcal{L}(\btheta, \bphi; \bx^{(i)}).
\end{equation}
$$

实验发现只要小批量样本数 $M$ 取得稍微大一点 (例如令 $M = 100$), 那么采样次数 $L$ 可以取得非常低 (例如 $L = 1$), 并且仍然能够取得比较好的效果.

## 参考资料

- <a href="https://arxiv.org/abs/1312.6114" target="_blank">[1312.6114] Auto-Encoding Variational Bayes</a>
- <a href="https://en.wikipedia.org/wiki/Variational_Bayesian_methods" target="_blank">Variational Bayesian methods - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Bayesian_inference" target="_blank">Bayesian inference - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Evidence_lower_bound" target="_blank">Evidence lower bound - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Jensen%27s_inequality" target="_blank">Jensen's inequality - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Convex_function" target="_blank">Convex function - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence" target="_blank">Kullback–Leibler divergence - Wikipedia</a>
- <a href="https://stats.stackexchange.com/questions/335197/why-kl-divergence-is-non-negative" target="_blank">information theory - Why KL divergence is non-negative? - Cross Validated</a>
- <a href="https://en.wikipedia.org/wiki/Gibbs'_inequality" target="_blank">Gibbs' inequality - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Monte_Carlo_method" target="_blank">Monte Carlo method - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Law_of_large_numbers#Strong_law" target="_blank">Law of large numbers - Wikipedia</a>
- <a href="https://stats.stackexchange.com/questions/60680/kl-divergence-between-two-multivariate-gaussians" target="_blank">mathematical statistics - KL divergence between two multivariate Gaussians - Cross Validated</a>
- <a href="https://en.wikipedia.org/wiki/Estimation_of_covariance_matrices#The_trace_of_a_1_.C3.97_1_matrix" target="_blank">Estimation of covariance matrices - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Multivariate_normal_distribution" target="_blank">Multivariate normal distribution - Wikipedia</a>
- <a href="https://math.stackexchange.com/questions/1524489/whats-the-domain-of-the-sample-average-in-strong-law-of-large-numbers" target="_blank">probability - Whats the domain of the sample average in Strong Law of Large Numbers? - Mathematics Stack Exchange</a>
- Bertsekas, Dimitri, and John Tsitsiklis. _Introduction to Probability_. 2nd ed. Athena Scientific, 2008. ISBN: 9781886529236.
- <a href="https://yunfanj.com/blog/2021/01/11/ELBO.html" target="_blank">ELBO — What & Why | Yunfan's Blog</a>
- <a href="https://en.wikipedia.org/wiki/Importance_sampling" target="_blank">Importance sampling - Wikipedia</a>

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

尽管自编码器通过无监督地学习数据空间与潜在变量空间之间的映射来有效地对输入数据进行压缩并对潜在变量进行重建, 但这种压缩和重建均是建立在确定性的映射的基础上, 无法用于生成任务. VAE 继承了自编码器的设计哲学并将其推广至生成场景下, 其对自编码器框架做出的最大改进就是结合了**变分推断** (Variational Inference) ("变分自编码器" 中的 "变分" 指的就是变分推断). 变分推断试图解决的问题是在一个参数化的概率函数族中如何逼近某个目标概率分布函数, 为此提出了**证据下界** (Evidence Lower Bound, ELBO) 并将原来关于函数族的优化问题转化为关于 ELBO 优化问题. ELBO 可重写为一个 K-L 散度项 (即正则化项) 与一个重建误差项的和的形式. 在 VAE 给出的假设下, ELBO 中的 K-L 散度项能够转化为可精确计算的闭合形式, 同时 VAE 还使用**重参数化** (Reparameterization) 方法解决了 ELBO 中的重建误差项的梯度传递问题, 从而顺利将变分推断移植到了自编码器框架下. 实验证明 VAE 能够十分出色地完成图像的生成任务, 生成的图像更加逼近原始图像数据.

接下来的小节里首先会讨论基础的 K-L 散度和蒙特卡洛方法等预备知识, 然后深入讲解变分推断的脉络与背后的缘由, 最后再在这些知识的基础上合拢得到 VAE.

## K-L 散度

K-L 散度, 全称为 **Kullback-Leibler 散度** (Kullback-Leibler Divergence), 是一种衡量两个概率分布之间差异的**非对称**度量, 一般用于在使用某个分布 $Q$ 逼近某个真实分布 $P$ 时需要对这两个分布之间的差距进行估计的场合下. 在探讨 K-L 散度之前首先介绍一下什么是信息熵和交叉熵 (K-L 散度也被称为 "相对熵").

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

实践中常常将交叉熵用于衡量 $P$ 和 $Q$ 这两个分布函数之间的差异 (其中 $P$ 代表真实 (目标) 分布, $Q$ 代表某个用于近似 $P$ 的参数化的分布), 这一点与交叉熵和 K-L 散度之间的关系有关.

### K-L 散度及其性质

离散情况下, 两个分布函数 $P$ 和 $Q$ 的 K-L 散度定义为

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
D_{\text{KL}}(p(x) || q(x)) \triangleq \mathbb{E}_{x \sim p(\cdot)}\left[\log\frac{p(x)}{q(x)}\right],
$$

其中 $p$, $q$ 为概率密度函数.

K-L 散度满足如下性质:

1. 非负性, 即 $D_{\text{KL}}(P \| Q) \geq 0$, 当且仅当 $P \equiv Q$ 时取等号.
1. 非对称性, 即 $D_{\text{KL}}(P \| Q) \not\equiv D_{\text{KL}}(Q \| P)$.
1. 两个 (多元) 高斯分布的 K-L 散度存在闭合形式的计算公式.

性质 1 和性质 3 将在下面给出详细证明. 关于对称性, 容易验证一个反例为两个成功概率分别为 $\frac{1}{2}$ 和 $\frac{1}{3}$ 的伯努利试验.

#### 非负性证明

易知 $\forall x > 0$,

$$
\ln x \leq x - 1,
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

由于两个概率分布 $P$ 和 $Q$ 的 K-L 散度降至为零当且仅当这两个分布相同, 将 K-L 散度作为两个概率分布之间差异的度量是十分自然的.

#### 与交叉熵之间的关系

K-L 散度与交叉熵之间关系密切. 实际上两个概率分布 $P$ 和 $Q$ 之间的交叉熵恰由 $P$ 的信息熵与 $P$, $Q$ 之间的 K-L 散度构成,

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
H(P, Q) = H(P) + D_{\text{KL}}(P || Q).
\end{equation}
$$

由 K-L 散度的非负性,

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

由 \eqref{eq:cross-geq-entropy} 式可知 $P$ 与 $Q$ 之间的交叉熵总是大于 $P$ 的信息熵, 并且差值正是 $P$ 与 $Q$ 之间的 K-L 散度.

#### 高斯分布之间的 K-L 散度

前面提到两个 (多元) 高斯分布的 K-L 散度存在闭合形式的计算公式. 此性质的一个特例将会在推导 VAE 下的 ELBO 时发挥作用, 这里首先给出完整推导. 已知服从多元高斯分布 $\mathcal{N}(\mu, \Sigma)$ 的随机向量 $\boldsymbol{X}$ 的概率密度函数为

$$
\begin{equation}
\frac{1}{\sqrt{(2\pi)^k|\Sigma|}} \exp{\left( -\frac{1}{2}(\xbb{x} - \xbb{\mu}^T) \xbb{\Sigma}^{-1} (\xbb{x} - \xbb{\mu}) \right)},
\end{equation}
$$

其中 $k$ 为维数. 于是任意两个多元高斯分布 $\mathcal{N}(\mu_1, \Sigma_1)$ 和 $\mathcal{N}(\mu_2, \Sigma_2)$ 之间的 K-L 散度

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

**蒙特卡洛方法**是一类通过随机采样来估计数学表达式的数值结果的计算方法. 在VAE中, 蒙特卡洛方法常用于近似ELBO中的期望值计算.

蒙特卡洛方法 (Monte Carlo methods) 指一类算法, 这类算法的思想是为所希望求得的目标数值精心设计试验, 通过重复多次随机抽样所得到的均值近似该试验的期望并间接近似目标数值. 蒙特卡洛方法的名称来源于摩纳哥 (Monaco) 的一家名为 Monte Carlo Casino 的赌场. 该方法的创始者, 物理学家 Stanislaw Ulam, 借鉴了他叔叔在这家赌场里赌博时的习惯从而发明了蒙特卡洛方法.

在蒙特卡洛方法的应用中最经典的一个例子是通过随机撒点来计算 $\pi$ 的值:

<figure class="align-center">
  <div style="display: inline-block;">
  <a href="https://en.wikipedia.org/wiki/Monte_Carlo_method#Overview">
    <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/d/d4/Pi_monte_carlo_all.gif/290px-Pi_monte_carlo_all.gif" alt="蒙特卡罗方法用于近似 $\pi$ 值" style="zoom: 100%; border: 2px solid #eef2f5;" />
  </a>
  </div>
  <figcaption>蒙特卡罗方法用于近似 $\pi$ 值</figcaption>
</figure>

这个例子中蒙特卡洛方法通过重复多次随机撒点并计算点落在扇区内的频率来近似期望 $\frac{\pi}{4}$, 从而间接近似所希望求得的值, 即 $\pi$.

蒙特卡洛方法的重点在于如何设计试验使得该试验的期望与所希望求得的值之间存在能够进行转换的数值关系. 蒙特卡洛方法的理论保证是[柯尔莫哥洛夫强大数定律](https://en.wikipedia.org/wiki/Law_of_large_numbers#Strong_law). 柯尔莫哥洛夫强大数定律的内容是独立同分布的随机变量的均值将以概率 1 趋近于期望.

## 变分推断与 ELBO

变分推断 (Variational Inference) 的名称来源于变分法 (calculus of variations), 变分法是数学分析的一个领域, 这一领域所要解决的问题是对泛函 (即一个将函数映射至实数的函数, 通常表示为函数及其导数的定积分) 进行优化, 解决方法是使用函数和泛函中的**微小变化**来寻找泛函的极大值和极小值, 此即 "变分".

- 变分推断的目的是通过优化泛函找到一个最优的近似后验 $q(z|x) \approx p(z|x)$.

1. **近似后验**: 变分推断引入一个来自较简单分布族的近似后验分布 $q(z|x)$, 在计算上更易处理. 目标是使 $q(z|x)$ 尽可能接近真实的后验分布 $p(z|x)$.

2. **优化目标**: 近似后验 $q(z|x)$ 和真实后验 $p(z|x)$ 之间的接近程度通过 Kullback-Leibler (KL) 散度来衡量:

   $$
   \mathrm{KL}(q(z|x) \| p(z|x)) = \int q(z|x) \log \frac{q(z|x)}{p(z|x)} \, dz
   $$

   然而, 直接最小化这个 K-L 散度也是不可解的, 因为它涉及真实后验分布.

3. **证据下界 (ELBO)**: 取而代之的是, 我们最大化证据下界 (ELBO), 这是一种可解的目标函数, 提供了对对数边际似然 $\log p(x)$ 的下界:

   $$
   \log p(x) \geq \mathbb{E}_{q(z|x)}[\log p(x, z) - \log q(z|x)]
   $$

   ELBO 可以重写为:

   $$
   \mathbb{E}_{q(z|x)}[\log p(x|z)] - \mathrm{KL}(q(z|x) \| p(z))
   $$

   其中, 第一个项是期望对数似然 (重构项), 第二个项是近似后验和先验之间的 K-L 散度.

4. **优化**: 通过最大化 ELBO, 我们同时:
   - 鼓励近似后验 $q(z|x)$ 生成能够很好解释观测数据 $x$ 的潜在变量 $z$ (通过期望对数似然项).
   - 确保 $q(z|x)$ 保持接近先验分布 $p(z)$ (通过 K-L 散度项).

- ELBO 可以转化为如下形式:

  $$
  \mathbb{E}_{q(z|x)}[\log p(x|z)] - \mathrm{KL}(q(z|x) \| p(z)),
  $$

  其中第一项是对数似然的期望 (即重建项), 而第二项是近似后验与先验之间的 K-L 散度.

**证据下界** (ELBO, Evidence Lower Bound) 是变分推断中的一个关键概念. ELBO是对对数边际似然的下界, 通过最大化ELBO, 我们能够优化变分推断的目标, 使得近似分布 $q_\phi(z|x)$ 尽可能接近真实后验分布.

ELBO可以被拆分为两部分: 重建损失和KL散度. 这是由于以下原因:

- **重建损失**: 对应于 $\mathbb{E}_{q_\phi(z|x)}[\log p_\theta(x|z)]$, 衡量的是根据潜在变量 $z$ 重建输入数据 $x$ 的能力. 因此, 它被称为重建损失.

- **KL散度**: 对应于 $D*{KL}(q*\phi(z|x) || p*\theta(z))$, 衡量的是近似分布 $q*\phi(z|x)$ 与先验分布 $p_\theta(z)$ 之间的差异.

这种拆分使得优化过程更为直观, 其中重建损失保证生成样本的质量, KL散度保证潜在空间的结构合理.

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

上式等号右边前一项 K-L 散度在 VAE 将编码器和解码器限制为协方差矩阵为对角阵的高斯分布的前提下具有闭合形式的解, 可以实现精确计算; 后一项则需要使用蒙特卡洛方法进行近似, 即

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

回到 \eqref{eq:elbo2} 式, 由于已经约定 $q_\phi(z | x)$ 与 $p_\theta(z)$ 均具有高斯分布的形式, K-L 散度项可重写为

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

可以看到重写后的 K-L 散度项已经能够由编码器分布的期望 $\mu_\phi$ 和方差 $\sigma_\phi^2$ 显式计算得到.

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
  1. 每轮迭代从训练集中抽取 $M$ 张图片构成的 mini-batch. 对于第 $m$ 张图片 $x^{(m)}$, 首先计算 K-L 散度, 然后从编码器中采样得到 $L$ 个 $z$ 计算重建损失, 并将 K-L 散度和重建损失相加得到第 $m$ 张图片的 ELBO 即损失函数值, 最后通过随机梯度下降法进行训练. 实验表明只要每轮随机梯度下降小批量数据集的大小足够, 即使 $L$ 设置得再小也无妨. 一般将 $L$ 设置为 1, 但需要确保 $M$ 不会太小.

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

---
title: 论文笔记《Scalable trust-region method for deep reinforcement learning using  Kronecker-factored approximation》
date: 2018-02-26 14:30:42
tags:
	- K-FAC
	- ACKTR
categories:
	- 论文笔记
description:
	- 将K-FAC用到Actor-Critic框架中的算法

---

本文分为两部分，首先介绍优化方法Kronecker-factored Approximate Curvature（K-FAC），然后再介绍强化学习算法Actor Critic using Kronecker-factored Trust Region（ACKTR）。

## K-FAC

- motivation
	+ Natural Gradient（NG）
		+ 假设我们想**求样本$x_1, \dots, x_n$所服从的概率分布**，我们的解决思路如下：
			1. 假设概率分布服从某一种参数化的概率分布（e.g., 假设$x \sim q(x ; \theta)$，其中，$q$为由$\theta$决定的某种分布（e.g., 均值为$\theta$，方差为1的高斯分布））；
			2. 建立对数似然函数$L(\theta) = \log \prod _i q(x_i ; \theta)$；
			3. 通过优化方法（e.g., SGD）找到$\theta = \mathop{\text{argmax}}_{\theta} \log \prod _i q(x_i ; \theta)$。
		+ 一般来说，在使用诸如SGD这种优化方法时，我们都会**避免较大的步长**，以保证收敛。因为梯度仅仅是一个局部信息而不是全局信息，也就是说仅当在当前点附近时，才能保证梯度方向是函数值增大的方向。
		+ 对于我们的最大化对数似然函数优化问题，我们**真正想要优化的“参数”**应该是分布$q$，而不是$\theta$，也就是说真正的优化问题应该是
		$$
		q = \mathop{\text{argmax}}_{q} \log \prod _i q(x_i).
		$$因此类比上面关于步长的讨论，我们希望迭代时相邻的两个分布$q_{\text{new}}$与$q_{\text{old}}$之间的“距离比较近”。
		+ 而$\theta$空间的距离跟$q$空间的**距离有可能是不一致**的，也就是说有可能尽管$\theta _{\text{new}}$与$\theta _{\text{old}}$很接近，但$q_ {\text{new}}$与$q _{\text{old}}$的“距离”却很远。在这种情况下，尽管$\theta$的step size很小，但由于$q$的step size很大，**收敛效果**并不理想。
		+ Natural Gradient就是解决上述问题的一种方案，它找到了一个方向$d$，使得当$\theta$沿着$d$方向更新时（i.e., $\theta _{\text{new}} = \theta _{\text{old}} + \alpha d$），$q(x ; \theta)$的变化不大，从而**控制了$q$空间的step size**。
	+ Levenberg–Marquardt（LM）
		+ NG能够保证$q(x ; \theta + d)$与$q(x ; \theta)$相近，但作为代价，**牺牲了部分的单调性**，也就是说不能保证对数似然函数$L(\theta + d) > L(\theta)$（记得当step size较小时，SGD能够保证单调性）。
		+ Levenberg-Marquardt则是解决上述问题的一种方案，它通过一个参数来**调整算法是更像NGD（$q$空间step size较小，收敛性质较好）还是SGD（目标函数单调递增/减）**。
	+ Kronecker-factored Approximate Curvature（K-FAC）
		+ 无论是NGD还是LM，都需要存储一个$DIM(\theta) \times DIM(\theta)$大小的矩阵$G(\theta)$，然后还要求解一个线性方程组$G(\theta)x=g$。在参数维度较高的问题中，这两种算法的**空间复杂度和时间复杂度都会非常的高**（e.g., 对于利用深度神经网络对$q$进行建模的算法，$O(\Vert \theta \Vert ^2)$的空间复杂度是不能接受的）。
		+ K-FAC针对神经网络这种结构，通过一系列的近似**降低了存储矩阵$G(\theta)$的空间复杂度以及求解线性方程组$G(\theta) x=g$的时间复杂度**，使得NGD和LM能够在神经网络上应用。
- idea
	+ NG
		- 求解一个方向$d$，使得当$\theta$沿着$d$方向更新时（i.e., $\theta _{\text{new}} = \theta _{\text{old}} + \alpha d$），$q(x ; \theta)$的变化不大，并且$L(\theta + d) > L(\theta)$，形式化来说就是求解以下优化问题：
		$$
		\begin{aligned}
		\mathop{\text{maximize }}_d & d^T \nabla _ \theta L(\theta) \\
		\text{subject to } & \Vert d \Vert _{G(\theta)} ^2 < \epsilon
		\end{aligned},
		$$其中，$\Vert d \Vert _{G(\theta)} ^2 = d^T G(\theta) d$，$G(\theta)$为Fisher Information Matrix。
			+ **约束条件**通过以symmetric KL divergence作为$q$空间“距离”的度量推导而来，体现了$\theta$沿$d$方向更新，$q(x;\theta)$的变化不大。具体来说，就是用$KL(q(x ; \theta) \Vert q(x; \theta + d))$来表示当$\theta$沿$d$方向更新时，$q$变化的大小；
			+ **目标函数**表示希望$d$与梯度方向一致，体现了$L(\theta + d) > L(\theta)$。
			+ 这里有个问题，既然把$d$理解成方向，为什么不加一个约束条件限制$d$为单位向量呢？
		- NG得到的$d \propto G(\theta)^{-1} \nabla L(\theta)$，$\theta$的迭代公式为$\theta _{k+1} = \theta _{k} + \alpha _k G(\theta)^{-1} \nabla L(\theta)$。
	+ LM
		- 将$d$改写为$d = (G(\theta) + \tau I)^{-1} \nabla L(\theta)$，通过调整$\tau$的大小来**调整算法趋向于NGD还是SGD**，具体来说，$\tau$小的时候趋向于NGD，$\tau$大的时候趋向于SGD。
		- 这里有个问题，调控$\tau$大小的算法中，$p$的含义是什么？
	+ K-FAC
		- 对**Kronecker product做近似**以降低$G(\theta)$的储存成本，再**强行对角化**（非对角block置为0）以降低求解方程组$G(\theta)x = g$的计算复杂度。
- more
	+ exponential average的迭代形式：
	$$G_{n+1} = (1-\epsilon) \hat{G}_n + \epsilon G_n \Leftrightarrow G_{n+1} = (1-\epsilon)( \hat{G}_n + \epsilon \hat{G}_{n-1} + \epsilon ^2 \hat{G}_{n-2} + \dots).
	$$
	+ K-FAC中优化参数变为$q(y \mid x ; \theta) \hat{q}(x)$；
	+ 针对LM的K-FAC（i.e., 近似$(G(\theta) + \tau I)^{-1}$的思路）。
- references
	+ [K-FAC tutorial](https://www.youtube.com/watch?v=FLV-MLPt3sU)
	+ [Kronecker product tutorial](https://hec.unil.ch/docs/files/23/100/handout1.pdf)

## ACKTR

- motivation
	+ 将K-FAC用到actor-critic框架中，提高sample efficiency，加快收敛速度。
- idea
	+ actor：对于stochastic policy，actor的输出就是分布，可以直接用K-FAC；
	+ critic：将回归问题
	$$
	\mathop{\text{minimize}}_{\theta} \sum_i \Vert f(s_i ; \theta) - v_i \Vert^2
	$$建模为最大似然问题
	$$
	\mathop{\text{maximize}}_{\theta} \log \prod _i p(v_i \mid s_i ; \theta).
	$$
	+ 针对sharing parameter framework：
		1. 理解为对pair(a, v)作最大似然估计；
		2. 假设actor与critic相互独立，得联合概率$p(a,v \mid s ; \theta) = \pi(a \mid s ; \theta)p(v \mid s ; \theta)$；
		3. 对$p(a, v ; \theta)$作最大似然估计。
	+ trust region：在K-FAC的基础上再对参数$\theta$更新的步长再做一个限制。
	
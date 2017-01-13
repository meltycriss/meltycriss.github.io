---
title: 论文笔记《Reciprocal Velocity Obstacles for Real-Time Multi-Agent Navigation》
tags:
  - VO(Velocity Obstacle)
  - RVO(Reciprocal Velocity Obstacle)
description:
  - 经典的底层避障算法
categories:
  - 论文笔记
date: 2017-01-10 21:08:51
---



## 简介

在介绍VO，RVO之前，需要先介绍路径规划。

对Agent进行路径规划，实际上要完成的**任务**就是让Agent从点A**无碰撞地**移动到点B。而路径规划的过程是层次化的，其基本框架大致如下：

- **High level**: dijkstra等算法。
- **Low level**: VO, RVO, ORCA等底层避障算法。

很容易可以跟我们的日常生活进行类比，比如说我们要从学校的教学楼走到宿舍楼，那么以上框架对应的就是：

- **High level**: 通过dijkstra算法，得到路径为: 教学楼→饭堂→体育馆→图书馆→宿舍楼。
- **Low level**: 通过底层避障算法如VO，RVO，ORCA等底层避障算法，保证我们走的每一段路（e.g. 教学楼→饭堂），都不会跟别的同学发生碰撞。

VO和RVO就是经典的底层避障算法。其中VO是最经典的，RVO则在VO的基础上进行了一些改进，解决了VO抖动的问题。

## VO(Velocity Obstacle)

**一句话总结**VO的思路：只要在未来有可能会发生碰撞的速度，都排除在外。

为方便描述，以下都假设是在**平面**内，**圆形物体**之间的避障。

## VO的直观理解

<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img prob_disc.png 问题描述 %}
</div>

> Q: 假设B静止，那么A取什么速度能够保证一定不会跟B发生碰撞呢？

> A: 一种很粗暴的方法，就是**把A化作质点**，选择跟**$\bar{B}$(扩展后的B)不相交**的速度方向。以后只要在每个周期里面，都选择不在VO的速度，就能够保证不会碰撞。

<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img prob_sol.png 解决办法 %}
</div>

以上就是VO的直观理解，需要注意的是：

- VO是指速度方向与$\bar{B}$相交的部分，即**会发生碰撞的部分（图中灰色斜线部分）**。
- VO是抱着**宁杀错，不放过**的思想，把所有未来有可能会发生碰撞的速度都放弃了。
- 实际上假如仅要求**一定时间内**不发生碰撞的话，有更多的速度可供选择，比如说上图中的($v\_{A}^{\prime}$)。

## VO的图示理解

有了直观理解之后就可以用更加严谨一点的数学语言图示VO了。

首先将直观理解中口语化的表达转换成对应的数学语言表示。

- **物体A(B)**：以$\bf{p\_{A}}$为圆心，$r\_{A}$为半径的**点集$A$**
- **假设B静止**：A相对于B的速度，即**相对速度$\bf{v\_{A}}-v\_{B}$**
- **把A化作质点**：求集合$B$与集合$-A$的Minkowski sum，即**闵氏和，$B\oplus-A$**，其中
	+ $A \oplus B = \\{    {\bf{a}} + {\bf{b}}\ |\ {\bf{a}} \in A, {\bf{b}} \in B \\} $
	+ $-A = \\{ -{\bf{a}}\ |\ {\bf{a}} \in A \\}$
	+ [更多关于Minkowski sum](http://twistedoakstudios.com/blog/Post554\_minkowski-sums-and-differences)

于是就有了下图的左半部分（浅色三角形）：

<div style="width:500px; margin-left:auto; margin-right:auto;" >
  {% asset_img vo.png VO %}
</div>

而为了直接求$\bf{v\_{A}}$绝对速度的VO而不是$\bf{v\_{A}}-v\_{B}$相对速度的VO，将相对速度下的VO延$\bf{v\_{B}}$方向平移，就有了图中右半部分（深色三角形）。

## VO的数学定义

理解了图示，数学定义就很好理解了。

- 首先给出**射线的定义**，用$\lambda({\bf{p}},{\bf{v}})$表示以点$\bf{p}$为顶点，方向为$\bf{v}$的射线。
$$\lambda({\bf{p}}, {\bf{v}}) = \\{ {\bf{p}} +t{\bf{v}} | t\ge 0\\}$$
- 接下来就是**VO的定义**了，用$VO^{A}\_{B}({\bf{v\_{B}}})$表示速度为$\bf{v\_{B}}$的$B$对$A$的VO 
$$VO^{A}\_{B}({\bf{v\_{B}}}) = \\{ {\bf{v\_A}} | \lambda({\bf{p\_{A}}}, {\bf{v\_{A} - v\_{B}}}) \cap B \oplus -A \ne \emptyset \\}$$

## RVO(Reciprocal Velocity Obstacle)

VO给出了很漂亮的避障条件，所以后面很多底层的避障算法都是基于VO的，而RVO就是其中之一。

RVO主要解决了VO的抖动问题

- **抖动现象**：如下左图所示，即$A$会在$\bf{v\_{A}}$与$\bf{v\_{A}^{\prime}}$之间来回切换
- **RVO的效果**：如下右图所示，保持$\bf{v\_{A}}$，不会抖动

<div style="width:700px; margin-left:auto; margin-right:auto;" >
  {% asset_img oscillation.png 抖动现象 %}
</div>

## 证明VO抖动现象存在

首先论文给出了VO的三条性质

- **Symmetry**：$\bf{v\_A}$的$A$会撞上$\bf{v\_B}$的$B$，则$\bf{v\_B}$的$B$也会撞上$\bf{v\_A}$的$A$
$${\bf{v\_A}} \in VO^{A}\_{B}({\bf{v\_{B}}}) \Leftrightarrow {\bf{v\_B}} \in VO^{B}\_{A}({\bf{v\_{A}}})$$
- **Translation Invariance**：$\bf{v\_A}$的$A$会撞上$\bf{v\_B}$的$B$，则$\bf{v\_A+u}$的$A$会撞上$\bf{v\_B+u}$的$B$
$${\bf{v\_A}} \in VO^{A}\_{B}({\bf{v\_{B}}}) \Leftrightarrow {\bf{v\_A+u}} \in VO^{A}\_{B}({\bf{v\_{B}+u}})$$
- **Convexity**：在$VO^{A}\_{B}({\bf{v\_{B}}})$的左（右）侧的两个速度之间的任意速度，也在$VO^{A}\_{B}({\bf{v\_{B}}})$的左（右）侧。VO左（右）侧如下图所示：
$${\bf{v\_A}} \overrightarrow{\notin} VO^{A}\_{B}({\bf{v\_{B}}}) \land  {\bf{v\_A^{\prime}}} \overrightarrow{\notin} VO^{A}\_{B}({\bf{v\_{B}}}) \Rightarrow (1-\alpha){\bf{v\_A}} + \alpha{\bf{v\_{A}^{\prime}}} \overrightarrow{\notin} VO^{A}\_{B}({\bf{v\_{B}}}),\ for\ 0\le \alpha \le 1$$

<div style="width:300px; margin-left:auto; margin-right:auto;" >
  {% asset_img left_right.png VO左（右）侧示意 %}
</div>

接下来是抖动现象存在的证明

1. 假设初始状态为会发生碰撞：${\bf{v\_A}} \in VO^{A}\_{B}({\bf{v\_{B}}}),\ {\bf{v\_B}} \in VO^{B}\_{A}({\bf{v\_{A}}})$
2. 由于在对方的VO内，所以各自选择新的速度以防止碰撞：${\bf{v\_A^{\prime}}} \notin VO^{A}\_{B}({\bf{v\_{B}}}),\ {\bf{v\_B^{\prime}}} \notin VO^{B}\_{A}({\bf{v\_{A}}})$
3. 由前面VO的Symmetry性质可知：此时，**原来的速度不在当前速度的VO内**：${\bf{v\_B}} \notin VO^{B}\_{A}({\bf{v\_{A}^{\prime}}}),\ {\bf{v\_A}} \notin VO^{A}\_{B}({\bf{v\_{B}^{\prime}}})$
4. 假设我们**更加prefer原来的速度**，则又会回到原来的$\bf{v\_A}$与$\bf{v\_B}$
5. 于是在**1→4之间循环**，即发生抖动

## RVO的Insight

首先回想一下为什么会发生抖动：

> 双方为了避障，都偏移了当前速度太多，导致更新速度后，原来速度不再会发生碰撞。

那么我们有没有办法减少对当前速度的偏移，同时又能保证避障呢，RVO的回答是肯定的：

- 缩小VO的大小，新的"VO"就叫做RVO
	+ p.s. 我个人对Reciprocal的理解是：相对于VO**完全把对方当做木头**，RVO假设对方在避障中也**会承担一定责任**，所以**不用完全靠自己**改变速度来走出VO，有种**互相合作**避障的感觉。
- 或者换一个角度理解，不再直接选择VO外的速度$\bf{v\_A^{\prime}}$作为新的速度，而是average当前速度$\bf{v\_A}$与VO外的速度$\bf{v\_A^{\prime}}$

## RVO的定义与图示

- 速度为$\bf{v\_{B}}$的$B$对速度为$\bf{v\_A}$的$A$产生的RVO为：

$$ RVO\_{B}^{A}({\bf{v\_B}}, {\bf{v\_A}}) = \\{ {\bf{v\_{A}^{\prime}}}\ |\ 2{\bf{v\_{A}^{\prime}}} - {\bf{v\_A}} \in VO^{A}\_{B}({\bf{v\_{B}}})\\}$$

- 图示理解如下：

<div style="width:500px; margin-left:auto; margin-right:auto;" >
  {% asset_img rvo.png RVO %}
</div>

- 释意：
	+ $2{\bf{v\_{A}^{\prime}}} - {\bf{v\_A}}$：${\bf{v\_A}}$相对于$\bf{v\_A^{\prime}}$的对称点。
	+ 所以**公式的含义**是：对称点在原VO中，则中点在RVO中。
	+ 所以**RVO的构成**是：$\bf{v_A}$与原VO中的点的中点。

## RVO不会发生碰撞且没有抖动现象的证明

这一部分不赘述了，论文中写得很详尽，只说一下证明的思路：

- 双方选择**同侧**避障时，不会发生碰撞。
- 双方一定会选择同侧避障。
- 不会有抖动现象：原来会撞的在选择新速度后**依然**会撞。

## 收获

- **用数学语言来描述问题**：化作质点的描述、抖动的描述。
- **从实际应用中发现问题**：抖动问题的发现。
- **特殊到一般的推广**：论文后面还将RVO推广到一般情况，很漂亮的推广。

## References

- [Reciprocal Velocity Obstacles for Real-Time Multi-Agent Navigation](http://gamma.cs.unc.edu/RVO/)
- [Minkowski Sums](http://twistedoakstudios.com/blog/Post554\_minkowski-sums-and-differences)

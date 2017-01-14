---
title: 论文笔记《Reciprocal n-body Collision Avoidance》
tags:
  - ORCA
description:
  - 应用广泛的经典底层避障算法
categories:
  - 论文笔记
date: 2017-01-14 21:41:51
---


## 简介

ORCA是经典的分布式底层避障算法，其任务是（对于A）：

- **输入**：A与B的形状（$r\_A$、$r\_B$）、位置（$p\_A$、$p\_B$）、速度（$v\_A$、$v\_B$），以及A的偏好量（$v\_A^{pref}$）和限制量（$v\_A^{max}$）。
- **输出**：不会发生碰撞的$v\_A^{\prime}$。
- **特点**：双方只要都采用ORCA，那么双方**无需通信**，**分布式**地求出来的$v\_A^{\prime}$与$v\_B^{\prime}$也不会发生碰撞。

跟RVO一样，ORCA也是基于VO的，但更加实用，因为：

- **效果上**：考虑了**速度的大小**，使得筛选粒度更细。不像VO和RVO只考虑速度方向。
- **效率上**：求解过程基本只用到了**线性规划**，比较高效。不像VO和RVO有大量的非线性求解。

## 引入时间窗口$\tau$

1. 如下图(a)，在**空间坐标**上，有
    + 位于$\bf{p\_B}$，半径为$r\_B$的$B$
    + 位于$\bf{p\_A}$，半径为$r\_A$的$A$
2. 如下图(b)，引入时间$\tau$，将**空间坐标**转换为**速度坐标**
    1. **相对化空间坐标**：将图(a)化为以$A$为原点，并将$A$化为质点
    2. **转为速度坐标**：假设$B$静止，那么当$v\_A * \tau$达到红色弧线时，$A$与$B$会发生碰撞，所以VO内，速度上限为$\frac{p\_{弧线上的点}}{\tau}$，得到绿色弧线。
    3. 至此得到$VO^{\tau}\_{A|B}$，即绿色弧线与两射线组成的类扇形
3. 如下图(c)，**考虑$B$取的速度集合$V\_B$**，对假设$B$静止得到的$VO^{\tau}\_{A|B}$求闵氏和，得$CA^{\tau}\_{A|B}(V\_B)$

<div style="width:800px; margin-left:auto; margin-right:auto;" >
  {% asset_img time_interval.png 时间窗口 %}
</div>

下面给出各变量规范的数学定义：

- $D({\bf{p}},r)$：以$p$为圆心$r$为半径的圆
 $$D({\bf{p}},r)=\\{ \ {\bf{q}} \ | \ \\| {\bf{q}} - {\bf{p}} \\| < r \ \\}$$

- $VO^{\tau}\_{A|B}$：绿色弧线和两条射线组成的类扇形
 $$VO^{\tau}\_{A|B} = \\{ \ {\bf{v}} \ | \ \exists t \in [0, \tau], t {\bf{v}} \in D({\bf{p\_B}} -{\bf{p\_A}}, r\_A+r\_B) \  \\}$$

- $CA^{\tau}\_{A|B}(V\_B)$：考虑$B$取速度集合$V\_B$
 $$CA^{\tau}\_{A|B}(V\_B)=\\{ \ {\bf{v}} \ | \ {\bf{v}} \notin VO^{\tau}\_{A|B} \oplus V\_B \ \\}，　其中\oplus为闵氏和$$

至此，成功在VO的基础上引入了时间窗口，避障的时候不仅考虑**速度方向**，还考虑**速度大小**。

p.s. 关于VO，闵氏和等内容可以参考上一篇文章{% post_link paper-rvo 论文笔记《Reciprocal Velocity Obstacles for Real-Time Multi-Agent Navigation》 %}

## ORCA定义

ORCA全称为Optimal Reciprocal Collision Avoidance，含义就是在上述的$CA^{\tau}\_{A|B}(V\_B)$中选择最优的一个。

论文对最优的定义大概可以理解为**在$v^{opt}$附近的合法值越多越好**，即为满足下列条件的$V\_A$与$V\_B$对：

- **Reciprocally collision avoiding**：很好理解，在对方的$CA$内即可，即
$$V\_A \subseteq CA^{\tau}\_{A|B}(V\_B) \land V\_B \subseteq CA^{\tau}\_{B|A}(V\_A)$$
- **Reciprocally maximal**：两个速度集是最大的不碰撞速度集，$V\_A$或$V\_B$都没有更多的避撞速度可选
$$V\_A = CA^{\tau}\_{A|B}(V\_B) \land V\_B = CA^{\tau}\_{B|A}(V\_A)$$
- **包含最多$v^{opt}$附近的避撞速度**：$v^{opt}$是一个预设值，可以理解为偏好值，论文中的最优希望：
	+ $v^{opt}$附近的值越多越好
- **双方在各自$v^{opt}$附近的避撞速度数量一样**：$A$和$B$各自有$v^{opt}\_A$和$v^{opt}\_B$，论文中的最优希望：
	+ $A$求出来在$v^{opt}\_A$附近的避撞速度的数量$n\_A$
	+ $B$求出来在$v^{opt}\_B$附近的避撞速度的数量$n\_B$
	+ $n\_A = n\_B$

那么用数学语言来描述上面几条性质就是：
$$| ORCA^{\tau}\_{A|B} \cap D({\bf{v^{opt}\_A}}, r) | = | ORCA^{\tau}\_{B|A} \cap D({\bf{v^{opt}\_B}}, r) | \ge min(|V\_A \cap D({\bf{v^{opt}\_A}}, r)|, |V\_B \cap D({\bf{v^{opt}\_B}}, r)|)$$

其中

- $ORCA^{\tau}\_{A|B}$与$ORCA^{\tau}\_{B|A}$：表示要求的最优速度集对，且满足reciprocally maximal，即
$$ORCA^{\tau}\_{A|B} = CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A}) \land ORCA^{\tau}\_{B|A} = CA^{\tau}\_{B|A}(ORCA^{\tau}\_{A|B})$$
- $V\_A$与$V\_B$：表示任意reciprocally collision avoiding的速度集对，即
$$V\_A \subseteq CA^{\tau}\_{A|B}(V\_B) \land V\_B \subseteq CA^{\tau}\_{B|A}(V\_A)$$
- $r$：表示了“附近”的大小
- $D({\bf{v^{opt}}},r)$：表示$v^{opt}$附近的速度

## ORCA求解

论文采用构造法求解出满足上述性质的ORCA，下面先说怎么做，再说为什么这样做就能满足性质。

下面从$A$的角度介绍构造过程（图示如下），这里假设$A$知道$\bf{v^{opt}\_B}$

1. 求解**相对速度**${\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}$
2. 求解$VO^{\tau}\_{A|B}$**边界上**离相对速度${\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}$**最近的点**$\bf{x}$
3. 求解从相对速度${\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}$指向$\bf{x}$的向量$\bf{u}$
4. 求解与${\bf{v^{opt}\_A}} + \frac{1}{2}{\bf{u}}$垂直的直线，将空间分为两个平面
5. 直线${\bf{v^{opt}\_A}} + \frac{1}{2}{\bf{u}}$指向的平面即为所求$ORCA^{\tau}\_{A|B}$

<div style="width:600px; margin-left:auto; margin-right:auto;" >
  {% asset_img orca_sol.png ORCA求解 %}
</div>

下面逐条解释为什么符合性质：

- **Reciprocally collision avoiding**：
	+ 由构造过程可知${\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}+{\bf{u}}$不在$VO^{\tau}\_{A|B}$内
	+ $A$取新速度${\bf{v^{opt}\_A}} + \frac{1}{2}{\bf{u}}$
	+ $B$取新速度${\bf{v^{opt}\_B}} - \frac{1}{2}{\bf{u}}$
	+ 此时，相对速度为${\bf{v^{opt}\_A}} + \frac{1}{2}{\bf{u}} - ({\bf{v^{opt}\_B}} - \frac{1}{2}{\bf{u}}) = {\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}+{\bf{u}}$，所以不在$VO^{\tau}\_{A|B}$内
- **Reciprocally maximal**：
	+ 要证$ORCA^{\tau}\_{A|B} = CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A})$
	+ 先求$CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A})$：
		* 回忆之前引入时间窗口$\tau$部分，构造$CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A})$分两步
			1. 先将类扇形往${\bf{v^{opt}\_B}}-\frac{1}{2}{\bf{u}}$移动
			2. 沿直线$ORCA^{\tau}\_{B|A}$方向扫
		* 先将类扇形往$\bf{v^{opt}\_B}$方向平移（此时，类扇形中的${\bf{v^{opt}\_A}}-{\bf{v^{opt}\_B}}$到了$\bf{v^{opt}\_A}$，且$\bf{x}$与$\bf{v^{opt}\_A}$的中点在直线$ORCA^{\tau}\_{A|B}$上）
		* 别忘了还要将类扇形往$-{\frac{1}{2}\bf{u}}$方向移动（此时，$\bf{x}$刚好位于直线$ORCA^{\tau}\_{A|B}$上）
		* 最后再沿直线$ORCA^{\tau}\_{B|A}$方向扫，得$CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A})$
	+ 而显然$CA^{\tau}\_{A|B}(ORCA^{\tau}\_{B|A})$正是$ORCA^{\tau}\_{A|B}$
- **包含最多$v^{opt}$附近的避撞速度**：
	+ 选择了最近的$\bf{x}$进行“突围”，那“舍弃”的速度应该是最少的
	+ p.s. 这一点的证明不太严谨，有更好的证明欢迎留言探讨
- **双方在各自$v^{opt}$附近的避撞速度数量一样**：
	+ $\bf{v^{opt}\_A}$到$ORCA^{\tau}\_{A|B}$的距离等于$\bf{v^{opt}\_B}$到$ORCA^{\tau}\_{B|A}$的距离
	+ 定义“附近”用的是圆

最后给出ORCA的数学定义

$$ORCA^{\tau}\_{A|B} = \\{ \ {\bf{v}} \ | \ ({\bf{v}} - ({\bf{v^{opt}\_A}} + \frac{1}{2}{\bf{u}})) \cdot {\bf{n}} \ge 0 \  \\}$$

其中

- ${\bf{u}}$是最小偏移
$${\bf{u}} = (\mathop{argmin}\_{ {\bf{v}} \in \partial VO^{\tau}\_{A|B}}{\\| {\bf{v}} - ( {\bf{v^{opt}\_A}} - {\bf{v^{opt}\_B}} )  \\|}) - ({\bf{v^{opt}\_A}} - {\bf{v^{opt}\_B}})$$
- ${\bf{n}}$是跟${\bf{u}}$同向的法向量
- 用${\bf{v}} \cdot {\bf{n}} \ge 0$来表示一个平面

## ORCA基本用法

1. 对其他所有agents的$ORCA$求交（线性规划），再与自己可选速度求交集，得候选速度集$ORCA^{\tau}\_{A}$
$$ORCA^{\tau}\_{A}=D({\bf{0}}, {\bf{v^{max}\_A}}) \cap \mathop{\bigcap}\_{B \ne A}{ORCA^{\tau}\_{A|B}}$$
2. 在候选集中求解跟自己偏好速度最近的一个速度${\bf{v^{new}\_A}}$
$${\bf{v^{new}\_A}} = \mathop{argmin}\_{ {\bf{v}} \in ORCA^{\tau}\_A}{\\| {\bf{v}} - {\bf{v^{opt}\_A}} \\|}$$

<div style="width:600px; margin-left:auto; margin-right:auto;" >
  {% asset_img orca_app.png ORCA基本用法 %}
</div>

可见ORCA的求解，就是在$ORCA^{\tau}\_{A}$内，优化目标函数$\\| {\bf{v}} - {\bf{v^{opt}\_A}} \\|$，所以计算效率很高。

## 关于${\bf{v^{opt}}}$的选择

为方便阐述，假设${\bf{v\_A}} = {\bf{0}}$，这样$ORCA^{\tau}\_{A|B}$就只是从原点移$\frac{1}{2}{\bf{u}}$

讨论${\bf{v^{opt}}}$的选择跟$ORCA^{\tau}\_{A}$的关系

- ${\bf{0}}$
	+ 一定有解，如下图(c)。很好理解，这相当于假设了别人都静止，那只要自己也静止，必然有解。
	+ 密集情况下容易死锁，因为这样相当于只考虑位置信息，而不考虑速度信息
- ${\bf{v^{pref}}}$
	+ 密集情况下容易无解，如下图(b)。因为一般来说${\bf{v^{pref}}}$都比较大
	+ 而且实际上也不可能观察得到别人的${\bf{v^{pref}}}$
- ${\bf{v\_{curr}}}$
	+ 两者的折中

<div style="width:800px; margin-left:auto; margin-right:auto;" >
  {% asset_img opt.png opt选择 %}
</div>

## 无解情况的处理

基本思路跟线性代数里面的投影差不多：选择离合法速度最近的速度

$$ {\bf{v^{new}\_{A}}} = \mathop{argmin}\_{ {\bf{v}} \in D({\bf{0}}, {\bf{v^{max}\_A}})} {\mathop{max}\_{B \ne A}{\ d\_{A|B}({\bf{v}})}} $$

下面慢慢拆分来看

1. $d\_{A|B}({\bf{v}})$：表示${\bf{v}}$的“违规”程度。定义为${\bf{v}}$到ORCA分割线的有向距离，${\bf{v}}$在ORCA外则取正
2. ${\mathop{max}\_{B \ne A}{\ d\_{A|B}({\bf{v}})}}$：对每个速度${\bf{v}}$，取“违规”最多的值作为它的“违规程度”
3. $\mathop{argmin}\_{ {\bf{v}} \in D({\bf{0}}, {\bf{v^{max}\_A}})} {\mathop{max}\_{B \ne A}{\ d\_{A|B}({\bf{v}})}}$：在可行速度内，求”违规程度”最小的速度

## 收获
- 改进可以从以下角度考虑：
	+ **效果**：只考虑了方向，没有考虑速度
	+ **效率**：计算复杂度

## References

- [Reciprocal n-body Collision Avoidance](http://gamma.cs.unc.edu/ORCA/publications/ORCA.pdf)

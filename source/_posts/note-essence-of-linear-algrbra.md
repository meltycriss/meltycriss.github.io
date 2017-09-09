---
title: 课程笔记《线性代数的本质》
tags:
  - 线性代数的本质
description:
  - 形象、直观地理解线性代数
categories:
  - 课程笔记
date: 2017-09-9 11:15:46
---

## 向量

### 不同视角下的向量

- physics：箭头
- cs：数字列表（e.g. $ \begin{bmatrix} 1 \\\\ 2\end{bmatrix} $）
- mathematician： $ \vec{\bf{v}} $
	* 抽象理解，任何相加和数乘有意义的东西
	* physics和cs都只是其中一种实例，还可以是其他任意满足性质的东西（e.g. 函数）

### physics view与cs view的联系

- 桥梁：坐标系
- 对偶性（独立推导、一一对应）：表示、运算（加法、数乘）
- 作用
	* physics→cs：已知想要的图形变换效果，使用$ \begin{bmatrix} 1 \\\\ 2\end{bmatrix} $与计算机进行交流 
	* cs→physics：已知$ \begin{bmatrix} 1 \\\\ 2\end{bmatrix} $相关的等式，使用图形直观理解物理意义

## 线性组合、张成的空间与基

为什么称$a \vec{\bf{v}} + b \vec{\bf{w}}$是$\vec{\bf{v}}$与$\vec{\bf{w}}$的**线性**组合：固定$\vec{\bf{v}}$的条件下，任意取$\vec{\bf{w}}$，得到的是一条线

## 矩阵与线性变换

### Linear Transformation

- transformation
	* 相当于function（输入、输出）
	* 但之所以不用function，是因为transformation更有动感（由一个地方运动到另外一个地方）
	* 用网格线可视化（i.e. 画出网格线变换前后的位置）的时候，直观感受是空间发生变化
- linear
	* 直线仍然是直线、原点位置不变（i.e. 保持网格线平行并等距分布）

### 如何表示这个Linear Transformation

- 一般来说，描述一个变换，我们需要记下所有的输入、输出pair
- 但由于线性变换具有保持网格线平行且等距分布的性质
- 发现假如原向量$\vec{\bf{v}}$是$\hat{\bf{i}}$和$\hat{\bf{j}}$的线性组合（伸缩），那么新向量$\vec{\bf{v}}\_{new}$会是$\hat{\bf{i}}\_{new}$和$\hat{\bf{j}}\_{new}$相同的线性组合（伸缩）
- 因此仅需记录$\hat{\bf{i}}\_{new}$和$\hat{\bf{j}}\_{new}$即有**足够信息**表示该线性变换（假设知道输入向量关于原向量$\hat{\bf{i}}$和$\hat{\bf{j}}$的线性组合，通过$\hat{\bf{i}}\_{new}$和$\hat{\bf{j}}\_{new}$即可以求得输出向量）
- 【以上全部都只用到了图形化的解释，没有涉及坐标这个概念（i.e. 保存这个信息可以不用坐标，而是直接把新的箭头画出来，然后做箭头的加法和数乘）】
- 【下面为了用数学来表示，开始引入坐标的概念】
- 为了用坐标表示，需要选择一个基，于是选择原来的$\hat{\bf{i}}$和$\hat{\bf{j}}$作为基，之后的所有坐标都基于这个基
- 引入坐标后，只需记录$\hat{\bf{i}}\_{new}$和$\hat{\bf{j}}\_{new}$的坐标，就有足够的信息表示该线性变换
- 该坐标可以表示成矩阵的形式，因此**矩阵与线性变换具有对偶性**

### 线性组合、线性变换、向量的表示

- 注意，下面三者是独立的：
	* 向量的线性组合：$\vec{\bf{v}}$如何由$\hat{\bf{i}}$和$\hat{\bf{j}}$通过加法和数乘得到
	* 线性变换：具有保持网格线平行且等距分布性质的mapping
	* 向量的表示：引入一组基，使用一个数字列表表示沿各基的伸缩
- e.g.
	+ 向量的线性组合：$\bf{v} = 2\bf{i} + 3\bf{j}$
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img combination.png 向量的线性组合 %}
</div>
	+ 线性变换
		* 通过变换后有${\bf{v}}\_{new}$、${\bf{i}}\_{new}$和${\bf{j}}\_{new}$
		* ${\bf{v}}\_{new}$关于${\bf{i}}\_{new}$和${\bf{j}}\_{new}$的线性组合仍然为2和3
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img transformation.png 线性变换 %}
</div>
	+ 向量的表示：
		* $\bf{i}$和$\bf{j}$的坐标可以基于任意基S
		* ${\bf{i}}\_{new}$和${\bf{j}}\_{new}$的坐标可以基于任意基T
		* 最终${\bf{v}}\_{new}$的坐标基于基T
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img representation.png 向量的表示 %}
</div>
- 一般情况下，对于向量的表示，都直接选取$\bf{i}$和$\bf{j}$作为基，这样做的好处是
	+ 可以直接从$\bf{v}$的坐标知道线性组合
	+ 不用再为${\bf{i}}\_{new}$和${\bf{j}}\_{new}$选取新的基
	+ 最终的${\bf{v}}\_{new}$的坐标与$\bf{v}$基于相同的基
- 核心在于三点：
	+ 追踪用于表示$\vec{v}$的向量
	+ 线性组合
	+ 向量的表示

### $A\vec{\bf{x}}$的两种理解

- 结果上来看，就是得到一个新的向量（箭头）
- 由于该箭头使用坐标的形式表示的，那么肯定跟基相关
- 过程可以有两个理解（主要是针对坐标x的理解不同）：
	* 把A看成是线性变换的表示【有发生线性变换，向量的表示没变】：那么$A\vec{\bf{x}}$就可以理解为将在基1下的坐标x经过线性变换得到的在基1下的新坐标
	* 把A看成是基2【没有发生线性变换，只是向量的表示发生了变化】：那么$A\vec{\bf{x}}$就可以理解为在基2下的坐标x在基1下的坐标

## 矩阵乘法与线性变换复合

- 两个矩阵相乘：一个结合两种变换的变换
- 复合矩阵求解：追踪${\bf{i}}\_{new}$和${\bf{j}}\_{new}$
- 直观理解解释：
	+ 交换律（$AB \neq BA$）不成立
	+ 结合律（$(AB)C = A(BC)$）成立：复合变换AB等价于顺序进行变换B、A

## 行列式

- 动机：衡量线性变换使得空间伸缩了多少
- 含义：
	+ 大小表示：单位正方形/立方体的面积/体积伸缩的比例
	+ 正负表示：定向（右手法则）
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img det.png 行列式的含义 %}
</div>
- 等价：
	+ 行列式等于0
	+ 降维
- 直观解释$det(AB) = det(A)det(B)$：
	+ $det(AB)$的含义是求解复合变换$AB$使得体积伸缩了几倍
	+ 复合变换$AB$首先进行变换$B$使得空间伸缩了$det(B)$，然后进行变换$A$使得空间伸缩了$det(A)$倍
	+ 所以体积总共伸缩了$det(A)det(B)$倍

## 逆矩阵、列空间与零空间

### 矩阵的用途：求解线性方程组

- $A\vec{\bf{x}} = \vec{\bf{b}}$：找一个经过变换后等于$\vec{\bf{b}}$的向量
- $det(A) \neq 0$：有唯一解，一一对应
	+ 假如有一个变换，能使得$\vec{\bf{b}}$变到$\vec{\bf{x}}$
	+ 那么只要对$\vec{\bf{b}}$进行该变换即可得到要求的$\vec{\bf{x}}$
	+ 因为对于$\vec{\bf{x}}$来说，先进行变换$A$后再进行该变换等于没变
	+ 所以称该变换为$A$的逆
	+ 由此含义可知，以下等价
		* 逆存在
		* 没有降维
		* 行列式不等于0
- $det(A) = 0$：不一定有解，除非$\vec{\bf{b}}$恰好在降维所在空间（列空间），此时有无限多个解（e.g. 两个共线向量表示该方向上的向量）
	+ 行列式等于0
	+ 降维（e.g. 空间由面变成线）
	+ 逆不存在
		* 假如逆存在
		* 存在一个变换（函数）由线map到面
		* 矛盾（违反函数的定义）

### Rank

- 动机：衡量降维的程度（降到点/线/面）
- 含义：降维后的维度

### 零空间

- 动机：衡量有多少向量被映射到零向量（$A\vec{\bf{x}}=\vec{\bf{0}}$）
- 假如没有降维，只有零向量会被映射到零向量
- 假如降维，会有其他向量被映射到零向量
- 被映射到零向量的向量组成零空间
	+ 零空间有非零向量
	+ 降维
	+ 行列式等于0

## 非方阵

- $\begin{bmatrix} 1 & 2 \\\\ 2 & 2 \\\\ 3 & 3 \end{bmatrix}$：
	+ $\begin{bmatrix} １ \\\\ ０\end{bmatrix}$ →　$\begin{bmatrix} 1 \\\\ 2  \\\\ 3 \end{bmatrix}$
	+ $\begin{bmatrix} 0 \\\\ 1\end{bmatrix}$ →　$\begin{bmatrix} ２ \\\\ 2  \\\\ 3 \end{bmatrix}$
- $\begin{bmatrix} 1 & 2 & 3\\\\ 2 & 2 & 3 \end{bmatrix}$：
	+ $\begin{bmatrix} １ \\\\ ０ \\\\ 0\end{bmatrix}$ →　$\begin{bmatrix} 1 \\\\ 2\end{bmatrix}$
	+ $\begin{bmatrix} 0 \\\\ 1 \\\\ 0\end{bmatrix}$ →　$\begin{bmatrix} 2 \\\\ 2\end{bmatrix}$
	+ $\begin{bmatrix} 0 \\\\ ０ \\\\ 1\end{bmatrix}$ →　$\begin{bmatrix} 3 \\\\ 3\end{bmatrix}$

## 点积与对偶性

- 证明点积与顺序无关思路：
	+ 先证等长时顺序无关
	+ 再证不等长时顺序无关（标量相乘）
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img dot_product_order.png 点积顺序无关 %}
</div>
- 对偶性：一一对应
- 点积：
	+ 物理含义：投影长度再乘以被投影向量长度
	+ 数学含义
- 证明对偶型思路：
	+ 验证单位向量点积：
		+ 定义投影到单位向量$\vec{\bf{\hat{u}}}=\begin{bmatrix} x \\\\ y \end{bmatrix}$的投影长度这种线性变换
		+ 得到投影长度这种线性变换对应的矩阵为$\begin{bmatrix} x & y\end{bmatrix}$
		+ 运算结果恰好等于与一个单位向量点积的数学形式
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img dot_product_projection.png 单位向量点积等价于投影长度这种线性变换 %}
</div>
	+ 验证非单位向量点积：
		+ 对于一个任意长度的向量$\vec{\bf{u}}=k\vec{\bf{\hat{u}}}$
		+ 定义投影到其单位向量$\vec{\bf{\hat{u}}}=\begin{bmatrix} x \\\\ y \end{bmatrix}$的投影长度乘以其长度k这种线性变换
		+ 得到投影长度这种线性变换对应的矩阵为$k\begin{bmatrix} x & y\end{bmatrix}$
		+ 运算结果恰好等于与一个非单位向量$\vec{\bf{u}}$点积的数学形式
- 将一个向量倒过来：投影到这个向量的投影长度乘以这个向量的长度这种线性变换对应的矩阵

## 叉乘

- 叉乘：
	+ 向量：大小为两向量平行四边形面积，方向为右手定则的向量
	+ 数学含义
		* 二维：$det(\begin{bmatrix} \vec{\bf{u}} & \vec{\bf{v}} \end{bmatrix})$
		* 三维：$det(\begin{bmatrix} {\begin{bmatrix} i \\\\ j \\\\ k \end{bmatrix}} & \vec{\bf{u}} & \vec{\bf{v}} \end{bmatrix})$
- 证明思路
	+ 根据叉乘的定义，他是由$det(\begin{bmatrix} {\begin{bmatrix} i \\\\ j \\\\ k \end{bmatrix}} & \vec{\bf{u}} & \vec{\bf{v}} \end{bmatrix}) = Xi + Yj +Zk$得到的向量$\begin{bmatrix} X \\\\ Y \\\\ Z \end{bmatrix}$
	+ 要证明$\begin{bmatrix} X \\\\ Y \\\\ Z \end{bmatrix}$的大小为$\vec{\bf{u}}$和 $\vec{\bf{v}}$构成的平行四边型的面积，方向为右手定则
		* $Xi + Yj +Zk$的大小的物理意义有两个（i.e. 所找向量必须同时满足以下两个性质）
			+ $\begin{bmatrix} X \\\\ Y \\\\ Z \end{bmatrix}$与$\begin{bmatrix} i \\\\ j \\\\ k \end{bmatrix}$的点积
			+ $\begin{bmatrix} i \\\\ j \\\\ k \end{bmatrix}$、$\vec{\bf{u}}$和 $\vec{\bf{v}}$构成的平行六面体的体积
		* 当$Xi + Yj +Zk$大小为$\vec{\bf{u}}$和 $\vec{\bf{v}}$构成的平行四边型的面积，方向为右手定则时同时成立

## 基变换

### 向量表示（坐标）的变换

- 从基于A系的坐标得到基于B系的坐标
	+ 对于一个坐标为$\begin{bmatrix} 2 \\\\ 3\end{bmatrix}$的向量
	+ 我们可以假设这个坐标是基于$\vec{\bf{u}}$和$\vec{\bf{v}}$的（i.e. 2和3是向量关于$\vec{\bf{u}}$和$\vec{\bf{v}}$的线性组合）
	+ 那么假如想要得到这个向量基于$\vec{\bf{i}}$和$\vec{\bf{j}}$的表示，只需得到$\vec{\bf{u}}$和$\vec{\bf{v}}$基于$\vec{\bf{i}}$和$\vec{\bf{j}}$的表示（i.e. 从基于A系的坐标得到基于B系的坐标）
	+ 从之前关于矩阵的两个理解可知该变换是个矩阵，记为M
- 根据上述得到的矩阵M直接取得B系到A系的坐标变换
	+ 假设B系到A系的坐标变换矩阵为M'
	+ 那么对于A系下坐标为$\begin{bmatrix} 2 \\\\ 3\end{bmatrix}$的向量
	+ 先进行M变换将该坐标转换为B系下的坐标，再进行M'变换将B系下坐标转换为A系下坐标，得到的结果必然还是$\begin{bmatrix} 2 \\\\ 3\end{bmatrix}$
	+ 由此可得M'M是一个什么都不干的变换，所以M'是M的逆

### 变换表示（矩阵）的变换

- 对于同一个线性变换，基于不同的系有不同的表示
- 假设线性变换在标准系下的表示为M，要求该线性变化在系A下的表示
	* 一般来说，在表示一个变换的时候，希望输入的表示和输出的表示是基于同一个系的
	* 输入的表示取决于基
	* 输出的表示取决于变换后基的表示
	* 所以为了输入输出基于相同系，变换后基的表示也取表示输入的基
	* 因此矩阵里面的列向量要与输入基于相同的基
	* 所以下面第一步是将基统一
- A：将A系下的表示转换为标准系下的表示
- MA：在标准系下进行转换
- A'MA：通过A'将标准系下的向量转换为A系下的表示
- 因此A'MA也称M的【相似矩阵】

## 特征向量与特征值

- eigen vector：变换后方向不变的向量
- eigen value：变换后在对应eigen vector的伸缩量
- eigen basis：
	* 有可能不够eigen vector来张成整个空间
	* 假如有的话，好处是：
		+ 在该变换下，基只发生了伸缩
		+ 用该基表示变换的话可以是对角矩阵，因为变换后基只与变换前基的一项相关，其余全是0
		+ 对角矩阵算幂很爽
	* 应用：算A的幂
		+ A是标准系下某个线性变换的表示
		+ 假如A有eigen basis（i.e.　足够多的方向不变向量）
		+ 求A对应的线性变换在eigen　basis下的表示（形式为B'AB），且该表示必然为对角矩阵【矩阵的对角化】
		+ 在eigen basis下进行幂运算
		+ 最后再把向量在eigen basis下的表示转回在标准系下的表示

## 抽象向量空间

- 基本就是1中所说的mathematician view（i.e.　满足一些性质（定义了加法、数乘，且运算封闭等等）的东西，physics view和cs　view等等都是向量的具体化）
- 这里举了一个例子：
	+ 向量：多项式函数
	+ 线性变换：求导

## 收获

- 对线性代数有了更加直观、深刻的理解
	+ 理清了原先混淆的概念
	+　知道了为什么一些操作的物理含义是这样的
- 对偶性的idea很美
- 视频开始的句子很美

## References

- [线性代数的本质](http://space.bilibili.com/88461692#!/channel/detail?cid=9450)

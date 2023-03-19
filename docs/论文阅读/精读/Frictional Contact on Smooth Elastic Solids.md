---
title: Frictional Contact on Smooth Elastic Solids
date: 2022-12-05T11:19:19Z
lastmod: 2022-12-05T13:35:55Z
share: true
tags:
  - PaperNotes
  - Collision
  - PhysicalSimulation
  - Friction
---


# Frictional Contact on Smooth Elastic Solids

 #PaperNotes ​, #Collision ​, #PhysicalSimulation ​, #Friction ​

---

问题：

1. 可变性弹性体之间的
2. 摩擦接触力

Previous Work：

1. 摩擦在碰撞处理步骤中进行：基于**多边形**之间发生接触

    1. 分辨率取决于网格（*接触分辨率*）：分辨率影响了模拟结果（缺乏对于的内在属性的描述）
    2. 网格无法定义*内/外侧*
    3. 无法定义平滑的距离场
2. Stable Sticking
3. **隐式**变形曲面表示下的接触解决方法：

    1. MLS近似 - [Moving Least Square Approximation](mweblib://16598659927655)
    2. 局部核函数
4. 摩擦力传递

    1. Parallel Transport Approximation
5. 动力学和弹性的**变分公式**自然地包含接触约束，

    1. 这些约束被解决为具有线性不等式约束的牛顿迭代法（Newton-Raphson Method）
6. ......

> 摩擦的非光滑性：对于物体静止放在一起、停止运动非常重要

## Introduction

1. 曲面表示的非光滑性（$C^0$）导致了摩擦计算与分辨率有关，导致低分辨率下的错误行为。

    * 隐式曲面
    * 优点：

      * 提供平滑的势函数来定义不等式约束优化问题
2. Focus：将摩擦隐式集成在算法中，确保在每一个时间步的**结束**，库仑定律精确满足。

    1. 对于刚性固体而言，Contact和Friction可能不正交

Contribution:

1. 可变性体的隐式曲面建模。

    1. 利用无穿透的约束 => 解决非线性约束
    2. 不需要额外的碰撞检测
2. 求解隐式曲面上非线性摩擦接触的算法

    1. 不需要将摩擦局部线性化
3. 时间步细分的机制（Time-Splitting）

    1. 允许更大的时间步长？
    2. 摩擦力冲量计算集成到*弹性*求解中

## Related Works

模拟刚体、弹性固体、布料等的方法：

1. SPH（流体动力学）
2. MPM
3. PBD
4. FEM

### 隐式曲面描述。

1. HRBF：Hermite 径向基函数
2. MLS

* 相同 & 缺点：Global Solve（依赖于所有点）
* 不同：mls不需要 *linear solve*

改进点：（MLS基础上）使用了一个更加复杂的核函数。

> [Morse et al. 2005; Ohtake et al.2003; Öztireli et al. 2009]

### 摩擦

解决方法：

1. 基于 Penalty：正向计算力的大小

    1. 碰撞的（相对运动）方向上添加恢复性的惩罚（力）（已经穿透，添加Force，使得不穿透，no IPC）

        1. 快速（实时）；
        2. 稳定性问题、刚性问题；
    2. 和接触表面切向（相对）速度线性的粘性力

        1. *lack the ability to model the biphasic nature of dry friction.*
2. 基于 Constraint：作为约束，直接得到力，不依赖于其他信息。

    1. 可以直接得到能够产生相应位移的力。
    2. 效率高，但是碰撞的处理因为不等式约束的加入而非常复杂

大多数方法选择**近似**库伦摩擦定律，而非精确求解。

> An artifact of this approximation is that two solids with different weights will start sliding at different times on a gradually increasing slope.

本文：

* 单独的弹性力求解、单独的摩擦力求解
* Focus：

  * 保真度（真实度）
  * 可扩展性

## BackGround

* Dynamic PDE
* Frictional Contact Resolving Method.

### 广义坐标（Generalized Coordinates）

Configuration Space:（构型空间，描述自由度，Generalized Coordinates）

$$
\begin{aligned}
\text{Position}: \mathbf q(t) \in \mathbb R ^ m\\
\text{Velocity}: \dot{\mathbf q}(t)\\
\text{Mass}: \mathbf M \in \mathbb R ^ {m \times m}
\end{aligned}
$$

则：$x_i$ 是参数化后的，（物理空间`Physical Space`​）坐标$x_i(\mathbf q )\in \mathbb R ^ 3$

### 运动方程

$$
\mathbf{M} \frac {d \dot{\mathbf{q}} } { dt } = \mathrm{ b ( t, \mathbf q , \dot{\mathbf{q}})}
$$

* $\mathrm b$：合外力

由于合外力会突变/短时间内特别大（冲量大），导致数值不稳定。使用**冲量**，而非力：

$$
\mathbf M \Delta \dot{\mathbf {q}} = \mathbf f^{t + \Delta t} 
= \int_t ^{t + \Delta t} b(s, \cdot, \cdot)\mathrm ds
$$

这样的符号系统下，总冲量分解为弹性力冲量 + 广义摩擦力冲量。

$$
\mathbf f =\mathbf f_e + \mathbf r
$$

因此：

$$
\dot {\mathbf q }^{t+\Delta t} = \dot {\mathbf q}^t + \mathbf M^{-1}
(\mathbf f_e ^ {t + \Delta t} + \mathbf r^{t + \Delta t})
$$

$$
\Delta \mathbf q  = \dot {\mathbf q} ^{t + \Delta t}
$$

### 碰撞

速度从C-S到P-S的映射为$J_i$

$$
J _ i := \frac{\partial x_i}{\partial \mathbf q}\implies 
\dot x_i := v_i = J_i \dot{\mathbf q}
$$

进而若 $f_i$ 是 P-S 的冲量，则$J_i' f_i$是C-S的冲量

牛顿第三定律：

$$
f _ j = - f _ i
$$

则整个系统冲量：

$$
\mathbf f = J_i ' f _ i + J _ i ' f _ j = (J_i - J_j) ' f _ i
$$

定义：$\mathbf J\in \mathbb R ^ {3n \times m}$为一个全局的**Contact Jacobian** 矩阵，其中$n$是总接触的数目

$$
\dot {\mathbf q }^{t+\Delta t} = \dot {\mathbf q}^t + \mathbf M^{-1}
(\mathbf f_e ^ {t + \Delta t} + \mathbf J' r^{t + \Delta t})
$$

类似于MPM的思想：

1. 用基于顶点的网格来表示实体变形
2. 用平滑的隐式曲面来模拟碰撞边界

假设$r^{t + \Delta t}$已知：

$$
\dot {\mathbf q } ^{t + \Delta t} = \arg\min_{\mathbf u }\frac 1 2 \| \dot {\mathbf q} -\mathbf u \|_{\mathbf M}^2 + W (\mathbf q ^{t + \Delta t}) - \mathbf u' \mathbf J'r ^{t + \Delta t}
$$

* $\| u \|_{\mathbf M} ^ 2=u'\mathbf Mu$
* $W$ 弹性势能
* $r$ 表明接触力冲量

> 本文实验的 FEM 模型选择为 non-linear / Stable neo-Hookean.

最终的方程：

$$
\dot {\mathbf q} ^ {t + \Delta t} = \mathop{\mathrm {argmin}} _ { \mathbf u }
 \frac 1 2 \| \dot {\mathbf q} -\mathbf u \|_{\mathbf M}^2 + W (\mathbf q^t + \Delta t \mathbf u) - \mathbf u' \mathbf J'r ^{t + \Delta t}
$$

1. 文章目的就是求解上述方程！其中的$r$满足库伦摩擦定律
2. 摩擦力冲量需要迭代求解。

## Implicit Surfaces

用多边形（3/4）来进行碰撞处理是有问题的。（Figure2）

​![image](assets/image-20221205112018-a081ox2.png)​

要求：

1. 表面光滑，可以写出*势能函数*：提供不等式约束给优化器（碰撞）
2. 势能场对变形是可微的：产生平滑的滑动
3. Local：减少计算量
4. 在C-S和P-S之间的插值是可微的。`?`​

先前的：

1. SDF：不可微，导致收敛问题

实际上，势函数只需要是：

1. 有符号的（Signed）
2. 光滑的

从而：<u>**可以通过系数（Scaling）来提升收敛速度**</u>

选择：Hausdorff 距离来度量近似误差。

### 局部MLS势能

假设有一个区域，$\Omega(\mathbf q)\subset \mathbb R ^ 3$，参数化为$\mathbf q$，有平滑的边界为$\partial \Omega$。

注意到 $\partial \Omega$ 有一个平滑的法向量场，设$\mathcal S$是 $\partial \Omega$ 上的采样点，都有法向量对应：

$$
\forall s\in \mathcal S, \quad s\rightarrowtail n_s
$$

我们的目标是计算势能：

$$
\Psi:\mathbb R^3 \rightarrow \mathbb R\qquad \forall s, \Psi(s) = 0
$$

在每一个顶点上，定义：

$$
\psi (x;s) = n_s'(x -s)
$$

> 原文：这个$\psi$是任意的，只要满足条件。

使用基函数加权，（必须$C^1$连续，原文使用重心坐标$w$）

$$
\Psi(x) = \sum_{s \in N (x)} w(x;s) \psi (x;s)\qquad
\sum_{s\in N(x)} w(x;s) = 1
$$

使用 local compact weight function，来获得稀疏的导数。

#### Weight Function

利用之前的权重函数 [Most and Bucher 2005]，该函数最初用于无元素 Galerkin 方法中的插值。该权重函数定义为

​![image](assets/image-20221205112037-jqutzi2.png)​

1. $\varepsilon$ 控制了拟合程度，越小越接近网格模型的表面（不一定是真实表面）；
2. $R$ 控制 $N_R(x) :\{s : \|x -s \| _ 2 < R\}$；

当 $R$ 比较小的时候，也可以用：

$$
\tilde w _ r ^ {cubic} (r) := 1 - 3 \left(\frac r R\right)^2 + 2 \left(\frac r R\right)^3
$$

#### 离散化

目的：确定采样点及其法线在形变过程中的变化。

1. 弹性模拟：标准的*四面体*有限元
2. 采样点选择：网格表面-曲面三角形的重心，$n_s$就是三角形的法向量

#### Contact Jacobian

涉及摩擦时，需要将切向力冲量映射到C-S。

设 $Q_s$ 为最小旋转矩阵:

$$
Q_s(x) n_s = \nabla _ x \Psi (x)
$$

> 用来将采样点处的法向对齐到接触点的梯度方向。

从而有：

$$
J_i = \frac{\partial x_i}{\partial \mathbf q} = 
\frac{\partial x _ i }{\partial s}
\frac{\partial s}{\partial \bar p}
\frac{\partial p}{\partial \mathbf q}
$$

* 第一项计算为$w_R(x;s)Q_s(x_i)\delta[s \in N_R(x_i)]$
* $s$和 $p$ 的关系是采样点和P-S的Mesh顶点的关系，故为$\frac 1 3 I$（对于三角形顶点和其重心采样点）
* $p$ 和 $q$ 的关系类似于微分几何中$r(u,v)$的关系

#### 选择 R

为了保证覆盖（防止势能场出现“孔”导致不连续）

1. 大于网格中最大的三角形（外接圆）直径的一半
2. 自适应（可能没做）

## Frictional Contact

Focus: 库伦摩擦

### Contact Space

令：

1. $\Gamma _ C$ 表示所有接触点的索引集
2. 对一个接触点$y\in \mathbb R^3$，定义Contact Space Coordinate（类似于曲面上的正交活动标架）：

    1. $\mathrm y_N$ - 法向坐标
    2. $\boldsymbol{\mathrm y} _ T$ - 切向坐标

定义

$$
\begin{aligned}
B = [B_N | B_T]\\
(B_N) _ {i , i} = n_i\\
(B _ T) _ {i , i} = [t_i| n _i \times t_i]\\
\end{aligned}
$$

* $n$ 接触的法向
* $t$ 接触的切向
* $i\in \Gamma _ C$

从而：

$$
\mathrm y_N = B_N^{T}y\quad \boldsymbol {\mathrm y_T} = B_T^{T}y
$$

### Coulomb Friction Model

摩擦接触冲量定义为：

$$
K_\mu = \left\{
r = (r_N, \mathbf r_T)\in \mathbb R^ 3: r_N\ge 0, \mathbf r _ T \in \mu r_ND
\right\}
$$

* $D$ 是单位圆盘。

只有以下三种情况：

$$
\begin{aligned}
r = 0\quad v_N >0\\
r \in K_\mu \quad v =0\\
r \in \partial K_\mu-0\quad v_T \in \{\alpha r_T: \alpha < 0\}
\end{aligned}
$$

### Contact

首先不考虑摩擦：

$$
0\le r_N \quad v_N \ge 0
$$

* 且最多有一个非零（SFC条件）

加上KKT：

​![image](assets/image-20221205112101-hmnl855.png)​

* $r$即为$\lambda$（利用kkt的互补松弛性）

> We use the dual form in our exposition, since it explicitly lists inputs and outputs of the optimization problem; however, in our implementation, we aim to solve a constrained minimization.
>
> 1. 说明中使用Dual Form
> 2. 求解中仍然使用约束优化

#### 无穿透约束

> **仅仅检测物体之间的碰撞。**

* 无穿透约束即为势函数大于0。

简记为：

$$
\phi _{k, i} (q) = \Psi [q_k] (p_i), \forall p_i \in \mathcal V_k
\implies\boldsymbol \phi (q) \ge 0
$$

用来替换$v_N$项（具体原因很奇怪，我觉得应该是它的梯度的法向分量来替换，但是替换的结果是没有问题的）

​![image](assets/image-20221205112115-yy4e9zn.png)​

#### Friction

（基于先前的工作，）可以直接引入：

​![image](assets/image-20221205112123-avarety.png)​

#### MDP（Maximal Dissipation Principle）

（先前的工作）将求$r_T$转化为约束优化问题：

$$
r_T = \arg\max_{y\in \mu r_N \bar D}  -y\cdot v_T=- \mu r_N \frac{v_T}{\|v_T\|}
$$

从而代入部分变量：

$$
\mathbf r_T = \mathop{\mathrm {argmin}}_{y \in D_{\Gamma_C} (\mu r_N)} y ' B_T' J\dot q
$$

​![image](assets/image-20221205112134-quln22u.png)​

### 求解方法

方程是（17）和（MDP）。

退而求其次，求解一个全局最优解的近似。

#### 时间细分：Predictor-Corrector

**Step1**：（Predictor）计算速度、法向接触力、

​![image](assets/image-20221205112145-0i0r2at.png)​

**Step2**:（Guess）用$r_N^*$作为Initial Guess，求约束力

​![image](assets/image-20221205112151-j7osr2r.png)​

**Step3**：（Correct）更新速度：

​![image](assets/image-20221205112156-29ddtq8.png)​

由于Step1内有一个非线性函数，可以局部线性化（线性展开近似）：

$$
\phi \approx \frac{\partial \phi}{\partial q} u + \frac 1 {\Delta t} \phi
$$

#### MSP

$$
P_T(r_N;\xi):= \mathop{\mathrm{argmin}}_ { y \in D _ {\Gamma_C}(\mu r_N)}
\frac 1 2 \|
B_T y - \xi 
\|^2 _ { M _ e ^{-1}}
$$

$$
M_e^{-1} := JM^{-1} J'
$$

* 上式为 Inverse of the *effective mass*.

$$
P_N(\xi) := \mathop{\mathrm{argmin}}_{y \ge 0} \frac 1 2 
\|
B_N y - \xi
\| ^2 _ {M_e ^ {-1}}
$$

​![image](assets/image-20221205112224-cbohiu2.png)​

不同点：

1. 分别从法向和切向构造。

用柱坐标去除非线性约束：

​![image](assets/image-20221205112238-0s9crk7.png)​

## 最终算法

​![image](assets/image-20221205112232-v4e4g8r.png)​

## 测试

1. Houdini 18
2. Rust
3. Ipopt内点法包
4. MKL with PARDISO
5. NN-Lookup: R*-Tree(2020) with bulk loading

## Limitations

1. 自相交（有关去除Neighbor）
2. 性能

    1. 使用直接的线性求解器：对可扩展性有影响。
    2. 从而使用了非线性求解器。
    3. 依赖于 Delassus 算子的显式构造
3. 采样方法：和以往（基于多边形的）算法相比，需要更精确/更多的采样，来保持尖锐的特征。

    1. 不能直接处理尖角（相当于过了一层滤波，直接磨平了）
4. 不能收敛

Discussion：

1. Friction Forwarding：在下一时间步的弹性求解中，引入了摩擦项

    ​![image](assets/image-20221205112247-1iusucd.png)​
2. 灵活性：

    1. 不仅仅依赖于多边形网格，也可以用于点云
    2. 方法与求解器算法无关

---
share: true
---

流固耦合解决的主要问题是流体与其他物体的交互。从固体是否能影响流体的运动、固体是否是刚体两方面，可以将常用流固耦合方法分为单向耦合、弱耦合、强耦合、浸没边界法四大类。单向耦合方法适合粒子与流体的耦合；弱耦合是图形学中最常用的方法，在一个时间步内先后解算流体与固体，需要较小的时间步长；强耦合在一个时间步同时进行流体与固体求解，来保证其不产生分离或穿透现象；浸没边界法是CFD中常见的方法，需要较大的计算开销，并没有在图形学中广泛应用。

- 处理固定速度（人为指定）的刚体墙：使用Chapter 5（Incompressibility）中给出的边界条件进行求解；
- 本章主要处理需要参与物理解算的物体

## One-Way：单向

- 假设流固边界的速度完全由流体决定（固体可以忽略）；
- 特别适合于轻、小物体与流体耦合；

$$
\frac{d\vec{x}}{dt}= \vec{u} (\vec{x}, t).
$$

也可以考虑流体对固体有一定的粘性（Net force: 净力）：

$$
F = - \iint _{\partial S} \tau \hat n
$$

如果是一个小物体 $i$，通常使用：

$$
F_{i}= D(\vec u - \vec u_{i})
$$

大一些的物体，可以使用散度定理，将面积分转化为体积分。

对于力矩

## Weak coupling

```python
# Algorithm: Weak Coupling
# Step 1: Advecting everything, i.e. x += x * h v
Advection()
UpdateSolidPosition()
# Step 2: Integrate non-coupled force (except pressure) to velocity (Splitting coupling, pressure, and other force, e.g. gravity, elastic)
IntegrateNonPressure();
# Step 3: Solve pressure, assume the solid velocity does not change within the time step
CopySolidVelocityToBoundaryCondition()
SolvePressure()
# Step 4: Update Solid velocity, due to Pressure, collision, etc.
CopyPressureToSolid()
SolveCollision()
ComposeFinalVelocity()
```

Core of Weak Coupling:

>One pressure solve for fluid treats the solid velocities as fixed, and one update to solid velocities treats the fluid pressure as fixed.

好处：便于实现，因为复杂的求解需要的计算公式和实现都是已有的。

`CopyPressureToSolid` 部分：依然是通过散度定理来转化为体积分

![[assets/Pasted image 20230417143951.png]]

吐槽一下第二行这个公式写的也太离谱了（画个图会死系列）：

![[assets/Pasted image 20230417145745.png]]

![[assets/Pasted image 20230417150048.png]]

缺点：并不是同时求解，因此物体的位置更新会迟一个Time Step，因此需要小的时间步长。

## IBM

Core：认为固体也是流体的一部分

![[../../其他教程/assets/Pasted image 20230416133607.png]]

一定程度上减轻Weak Coupling的问题。

> 1. https://zhuanlan.zhihu.com/p/586136608
> 2. https://faculty.cc.gatech.edu/~turk/my_papers/rigid_fluid.pdf
> 3. https://www.cambridge.org/core/services/aop-cambridge-core/content/view/95ECDAC5D1824285563270D6DD70DA9A/S0962492902000077a.pdf/div-class-title-the-immersed-boundary-method-div.pdf
> 4. https://www.fields.utoronto.ca/programs/scientific/10-11/fluid_motion/Lai.pdf

## Strong Coupling：强耦合

Notation For rigid body:

- 质心：$\vec X$
- 平动速度：$\vec V$
- 角速度 $\vec\Omega$
- 角动量 $\vec L$
- 质量$m$，转动惯量矩阵$I$

$$
\vec V^{n+1} = \vec V - \frac{\Delta t}{m}\iint _{\partial S} p\hat n
\qquad\vec \Omega^{n+1} = \vec {\Omega} - \Delta t I^{-1} \iint _{\partial S} (\vec{x} - \vec X) \times p \hat n
$$

Solid 上一点的速度是：

$$
\vec u_{solid}(\vec x) = \vec V+\vec\Omega \times(\vec x - \vec X)
$$

无粘流体、耦合求解方程为：

1. 不可压缩条件
2. 流固边界条件：在法向 $\hat n$ 上，固体速度(RHS，$n+1$步)和流体经过Pressure求解后速度（也是$n+1$步）相等。（当前的$u$只是Operator Splitting求解了前一半的结果）

$$
\begin{aligned}
\nabla \cdot \frac{\Delta t}{\rho} \nabla p &= \nabla \cdot \vec u\\
\left(\vec u - \frac{\Delta t}{\rho} \nabla p\right)\cdot \hat n &= \vec{u}^{n+1}_{solid}\cdot \hat n
\end{aligned}
$$

上面的式子需要进行简化，因为$\cdot \hat n$ 难算。

求解Pressure，转化为最小化总机械能。（实际上还有一部分是处理无散求解）

$$
E = \mathrm{KE}_{Solid}+\mathrm{KE}_{Fluid} + \frac{1}{2}\|Ap-\nabla \cdot u\|_{2}^{2}
$$

首先是原本的，求解无散压强的方程：

$$
A p = b
$$

并且利用 $J$ 的性质：

$$
J: p\mapsto  F,T
$$

左侧为刚体 $n+1$ 时刻的机械能，对 $p$ 求导：

![[assets/Pasted image 20230417162006.png]]

将 $\Delta t J^{T}M^{-1}J$ 加到 $A$ 上，将$-J^{T} U$加到RHS上。变量只有$p$

一个可能的优化是，求解如下的矩阵方程（更加稀疏）：

![[assets/Pasted image 20230417152247.png]]


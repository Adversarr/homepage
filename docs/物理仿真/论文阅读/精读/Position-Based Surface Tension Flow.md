---
share: true
tags:
  - PositionBasedDynamics
  - SurfaceTension
---


TOG 2022

> Abstract: 本文提出了在PBD框架下流体表面张力的计算方法，为PBD框架下的流体模拟提供了更真实的表面张力和与固体的接触，能够支持薄膜、液滴等复杂界面。其关键是动态地对流体表面进行局部网格划分。该算法通过局部网格划分，得到每个表面粒子周围的局部几何体，并基于该几何体，计算表面张力约束，整合入PBD框架中。相较于其他方法，该算法具有较高的计算效率较高和鲁棒性。

#PositionBasedDynamics, #SurfaceTension 

## Position based Dynamics

See [[PBD General]] for more details.

## Introduction

-   对于牛顿第二定律$M \ddot{\mathbf x} = \mathbf {f}_{ext}$和约束 $C(\mathbf x) \ge 0$
-   思路：先做显式时间积分，然后强制让积分结果满足约束
-   关键点：如何让结果满足约束？

![[Drawing 2023-03-15 14.24.09.excalidraw]]

### 约束满足算法

对于 $C(x)$ 进行线性近似：

$$
0 \le C(x+ \Delta x) \approx C(x) + (\nabla C(x) )^{T}\Delta x
$$

并适当选取 （动量守恒）

$$
\Delta x_{i} = \lambda \frac{1}{m_i}\nabla_{x_i} C(x)
$$

解出 

$$
\lambda  =\frac{C(x)}{\sum_{j} w_{j} |\nabla_{x_{j}} C(x)|^{2}}
$$

以及：

$$
\Delta x = - \frac{C(x)}{(\nabla C(x))^{T} \nabla C(x)}\nabla C(x)
$$

```python
for c in constraints:
  select = index(c)
  delta_x = project(c, x[select])
  x[select] += delta_x
```

![[Pasted image 20230315145831.png]]

### Example -- Mass spring

PBD 中的 「约束」 指代了软约束。

对于弹簧质点模型，弹簧在 Rest Pose 下满足：

$$
C(\mathbf x_{1}, \mathbf x_{2})= \| \mathbf x_{1} -\mathbf x_{2}\| - d = 0
$$

则有

$$
\nabla_{\mathbf x_{1}} C = \mathbf n = \frac{\mathbf x_{1} -\mathbf x_{2}}{\|\mathbf x_{1} -\mathbf x_{2}\|}
$$

并且

$$
\lambda = C(\mathbf{x}_{1}, \mathbf {x}_{2}) / 2
$$
从而：

$$
\Delta x_{1} = \frac{m_{2}}{m_{1}+m_{2}}C(\mathbf x_{1} ,\mathbf x_{2}) \mathbf n
$$

![[Pasted image 20230315151806.png]]

> Reference: A Survey on Position Based Dynamics, 2017 (EG 2017)

Algorithm:

```python
def project():
  for i in range(max_iter):
    for s in springs:
      x1, x2 = position[s.i], position[s.j]
      impulse = (norm(x1 - x2) - d) * normalize(x1 - x2)
      x1 += impulse * m2 / (m1 + m2)
      x2 -= impulse * m1 / (m1 + m2)
```

在这里，解释了为何[[#约束满足算法]]选取$\Delta x$时，公式中含有 $m$：

$$
m_{1}\Delta x_{1}+ m_{2}\Delta x_{2} = 0
$$

### Other Constraints

1. Stretching
2. Bending
3. Isometric Bending
4. Collision

## Position based Fluid

对于流体而言，其Constraint 表现为「不可压缩」，即对于所有的流体粒子$i$：

$$
\rho_{i}= \rho_{0}
$$

即：

$$
C_{i}(\mathbf p_{1},\cdots,\mathbf p_{n}) = \frac{\rho_{i}}{\rho_{0}}  - 1
$$

基于SPH的密度计算公式：

$$
\rho_{i}= \sum\limits_{j} m_{j} W(\mathbf{p}_{i} - \mathbf{p}_{j}, h)
$$

我们用 PBD 的公式做同样的推导：

$$
C(\mathbf p + \Delta \mathbf{p}) = 0
$$

其中

$$
\nabla_{\mathbf{p}_{k}}C_{i} = \frac{1}{\rho_{0}} W(\mathbf{p} _{i}-\mathbf{p}_{j}, h)
$$

代入求$\lambda$

$$
\lambda_{i}= - \frac{C_{i}(\mathbf{p}_{1} , \cdots \mathbf{p}_{n})}{\sum\limits_{k} | \nabla \mathbf{p}_{k}C_{i}|^{2}+ \varepsilon}
$$

从而，位置更新为

$$
  \Delta \mathbf{p}_{i}= \frac{1}{\rho_{0}}\sum\limits_{j} (\lambda_{i}+\lambda_{j}) \nabla W(\mathbf{p}_{i} - \mathbf{p}_{j}, h)
$$

![[Pasted image 20230315195643.png]]

## Position based Surface Tension

相同的思路，我们也是先给出Surface Tension对应的约束，然后用完全相同的方法来求位置更新。

![[Drawing 2023-03-16 14.56.36.excalidraw]]

![[Pasted image 20230315195840.png]]

### Previous Work

Lagrangian Surface Tension 主要的方法有三类：

第一类，不需要确定流体表面，通过计算**每一对**粒子间吸引力来模拟表面张力，能够保证动量守恒。缺点是依赖于粒子表示的精确程度，粒子少的地方精确性差，例如对于薄膜等不规则形状的流体（只有一层流体的情况），难以计算：

![[Pasted image 20230315200944.png]]

第二类，基于表面张力最小化表面积的特性，通过求出流体表面**局部**的法向和曲率，在法向施加力来最小化流体表面积。缺点是其依赖于一个高鲁棒性、精确的表面检测算法，来近似局部的几何形状。

第三类，建立表面网格，来计算更加精确的流体表面法向、曲率信息。优势在于，其能够进行精确的近似，并支持薄的、不规则的流体的模拟。缺点在于，该方法需要巨大的计算量用于生成精确的、高质量的网格，也不利于并行算法的实现。

![[Pasted image 20230315202845.png]]

### Contribution

我们依然基于 **表面张力→最小化流体表面积** 来定义约束。而最小化流体**总**表面积，可以通过最小化每一个表面粒子所在**局部**的表面积和实现。

问题：如何计算局部的表面积？

解决方法：分为以下四步（在找出表面粒子之后）

![[Pasted image 20230315210144.png]]

图中的 $\mathcal M_{i}$ 即为用于计算表面积的局部网格。

假设其中的三角形全体为集合 $T(i) = \{t_{1}, \cdots , t_{k_{i}}\}$

$$
  C_{i}^{A}(\mathbf{p}) = \sum\limits_{t\in T(i)}\frac{1}{2}\mathrm{Area}(\mathbf{p}_{t_{1}}, \mathbf{p}_{t_{2}},\mathbf{p}_{t_{3}})
$$

![[Pasted image 20230315211444.png]]

直接代入PBD公式计算得到更新位置即可。

为了解决约束导致的表面张力过大，导致粒子倾向于聚集在表面的问题，引入了另一个约束：

$$
C_{ij}^{D}(\mathbf{p}) = \min \{ 0, \| \mathbf{p}_{i}- \mathbf{p}_{j}\| - d_{0}\}
$$

![[Pasted image 20230315212225.png]]

其中的 $i, j$ 为同类型的粒子（表面、内部）。

### Details

#### 粒子分类为表面和内部

![[Pasted image 20230315212717.png]]

计算上图所示的$R_{illum}$，大于一定阈值的粒子即为表面粒子。

#### 法向计算和投影平面选取

基于SPH的公式来计算：

$$
c(\mathbf{p}) = \sum\limits_{j\in N(\mathbf{p})} \frac{m}{\rho_{j}} W(\mathbf{p} - \mathbf{p}_{j,}h)
$$

该函数取值为 0 时表明为流体外部（类似于Signed Distance Field）

其中的 $W(\mathbf{p}, h)$ 函数 图像如下：

![[Pasted image 20230315214204.png]]

因此

$$
-\frac{\nabla c(\mathbf{p}_{i})}{\|\nabla c(\mathbf{p}_{i})\|} \approx \mathbf{n}_{i}
$$
#### 局部网格构建

在得到投影后的点后，使用denaunay三角化算法构建三角网格。

选取三角网格中，点 $i$ 的 1-邻居为 $\mathcal M_{i}$

### Performance

![[Pasted image 20230315211719.png]]


### 优缺点分析

优点：

1. 每个表面粒子的计算相互独立，有助于并行算法的实现。
2. 对比于PBD Fluid - 支持薄膜的模拟
3. 对比 Explicit Force Methods - 鲁棒性高
4. 对比 Surface Area Calculation - 收敛性更高
5. 对比没有距离约束 - 解决粒子分布问题（图8）

缺点：

1. 依赖于精确的表面粒子检测
2. 不是物理准确的
3.  没有使用加速算法

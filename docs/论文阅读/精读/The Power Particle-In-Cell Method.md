---
title: The Power Particle-In-Cell Method
date: 2022-12-05T13:38:34Z
lastmod: 2022-12-10T14:38:49Z
share: true
tags:
  - MPM
  - PaperNotes
  - PhysicalSimulation
  - Fluid
---


# The Power Particle-In-Cell Method

#MPM ​, #PaperNotes ​, #PhysicalSimulation ​, #Fluid ​

> 原文：[sig22 P-PIC optimized.pdf](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf)
>
> Video: [Youtube](https://www.youtube.com/watch?v=fhq0HpAUkLY)

# 阅读

## Stage 1

> [第一遍，问题泛读，看Title，Abstract，Introduction，Subtitles 和 Figures。了解论文是做什么的（做什么问题）？Introduction中对这个问题的描述...](siyuan://blocks/20221205135741-5kwkb20)

### 研究背景和问题

流体仿真，特别是混合欧拉拉格朗日方法的流体仿真

1. 基础算法：PIC、FLIP
2. 当前MPM：可以用来仿真塑形形变、粘性、多维度物体

出现的问题：难以确保流体的不可压缩性。

> [sig22 P-PIC optimized.pdf - p2 - The archetypal advection-projection splitting between Lagrangian particles and Eulerian grids accumulates noticeable volume change over time that yields particle clumping and artificial voids, thus degrading the simulation accuracy and stability. ](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=2)

该问题先前的工作：

1. 后处理，添加类似于弹簧的力。（拉格朗日视角，Ando 2012）
2. 额外的压力求解。（欧拉视角，Kuglstadt 2019）

#### 工作基础 - The Power Particles method

> [sig22 P-PIC optimized.pdf - p2 - The Power Particles representation provides the optimal transportation plan we seek for mapping volume from particles to the continuum [Aurenhammer et al. 1998].](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=2)
>
> 笔记：[Power Particles: an incompressible fluid solver based on power diagrams](siyuan://blocks/20221206181613-2yz52t8)

问题：

1. 计算成本过高；
2. 仿真分辨率上缩放效果不佳

#### 本文工作

在 $P\rightarrow G$ 过程中，使用本文提出的权重，而非B-Spline等简单权重，来避免上述问题：

1. 使粒子均匀分布
2. 保持体积

主要贡献点

1. 重写 Power Particles：时间优化（Regularized Representation with volume-constrained density kernels） ：[定义和建模](siyuan://blocks/20221207101532-tei9qh5)
2. P→G 权重：体积保持、均匀分布： [Power Particle In Cell](siyuan://blocks/20221207111046-yhuil4i)
3. 处理自由表面（空气边界）：[Free Surface And Solid Obstacles](siyuan://blocks/20221207140708-lxqp0cm)
4. 计算加速：[计算加速](siyuan://blocks/20221207145454-bm1vfj2)

## Stage 2

> [第二遍，技术泛读，稍微看一下技术部分。了解论文是怎么做的？论文所用的技术方法、技术路线是否合理？是否非平凡的方法？是否有技术含量和技术贡献？必须认真看一下其limitation和future ...](siyuan://blocks/20221205135752-rmyv14d)

### 定义和建模

* 粒子下标为 $p$、Euler 网格为$i$、Transportation 网格为 $j$

#### Optimal Transportation

$T_{pj}$表示p → j 的体积权重。（LP问题）

$$
\min \sum_{p, j} T_{pj} \|x_p - x_j\|^2\\
s.t.\begin{cases}
\sum_p T_{pj} = V_j \forall j\\
\sum_j T_{pj} = V_p \forall p
\end{cases}
$$

问题：

1. 对于求解的定义域大小敏感
2. 高分辨率网格的求解速度慢

> [sig22 P-PIC optimized.pdf - p4 - Unfortunately, these strategies are known to scale poorly relative to the domain size due to the coupled particlecell unknowns, thus making the generation of transportation plans prohibitive for large particle count and high resolution grids.](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=4)

#### Entropic Regularization

正则化项

$$
\min \sum_{p, j} T_{pj} \|x_p - x_j\|^2 + \varepsilon \sum_{p, j} \mathcal {H} (T_{p, j})\\
s.t.\begin{cases}
\sum_p T_{pj} = V_j \forall j\\
\sum_j T_{pj} = V_p \forall p
\end{cases}
$$

可以变为凸问题：（$r$为拉格朗日乘子）

$$
T_{pj} = \psi^{\varepsilon} (r_j + r_p - \| x_p - x_j \| ^2) = s_{j}s_p K_{pj}
$$

#### Sinkhorn Iterations

数值优化上：ADMM

> [sig22 P-PIC optimized.pdf - p5 - We optimize these scaling unknowns using the iterative method of [Sinkhorn 1967] summarized in Algorithm 1 that alternates the update of each variable by enforcing its respective volume constraint. ](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=5)
>
> Algorithm：[sig22 P-PIC optimized.pdf - p5 - sig22 P-PIC optimized-P5-20221207105054](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=5)

#### The power kernel

Density Kernel:

$$
\chi_{p}^{\varepsilon} (x) = \psi^{\varepsilon}(r_p - \| x_p - x\|^2)/\sum _q\psi^{\varepsilon}(r_q - \| x_q - x\|^2)
$$

类似于[Radial Basis Function](siyuan://blocks/20221207110725-qns74ic)，但采用了Power Distance，因此称为 Power Kernel。

$$
\int_{\Omega} \chi_p ^\varepsilon \approx V_p
$$

#### Power Diagrams

利用[The power kernel](siyuan://blocks/20221207103626-fpw42zq) 的性质：

$$
\lim_{\varepsilon \rightarrow 0} \chi  = \begin{cases}
1, &\| x_p - x\|^2 - r_p \le \| x_q - x\| ^2 - r_q\forall p \ne q\\
0, &otherwise
\end{cases}
$$

几乎等价于Power Diagram的定义。因此，使用$\chi$来代替原本的距离函数。

#### Power Particles

see [Power Particles: an incompressible fluid solver based on power d...](siyuan://blocks/20221206181613-2yz52t8)

‍

### Power Particle In Cell

#### S-grid v.s. T-grid

插值基函数：$N_i(x)$  

s→t过程为：

$$
N_{ij} := N_i (x_j)
$$

实验中，使用了分片线性插值。

#### Power Weights

基于 [GIMP](siyuan://blocks/20221207133715-uiz6klw)：

> [sig22 P-PIC optimized.pdf - p6 - We construct our weighting scheme based on the GIMP method [Bardenhagen and Kober 2004; Gao et al. 2017], which defines weights by computing the correlation between particle-based characteristic functions and grid-based interpolants](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=6)

$$
w_{pi} = \frac{1}{V_p} \int_{\Omega} \chi_p^{\varepsilon}(x)N_i(x)\mathrm dx \approx \frac{1}{V_p} \sum_{j} T_{pj}N_{ij}
$$

> 加权再加权，得到P2G & G2P的权重。

### Free Surface And Solid Obstacles

#### Slack Air Variables

$a_j$，用于定义 cell-j 中有多少体积空气。

$$
V_j = \sum_{p} T_{pj} + a_j
$$

#### Air Volume 

### 计算加速

技术栈：

1. Eigen
2. VDB
3. SpVM
4. AMGCL

其他工作：

1. 优化空间数据结构。
2. 修正正则化项的影响。

    [sig22 P-PIC optimized.pdf - p8 - As pointed out by Cuturi [2013], the Sinkhorn algorithm converges faster as the regularization amount 𝜀 increases. However, larger values of 𝜀 end up creating a denser matrix for the Gaussian kernel and then the overall performance of each Sinkhorn iteration worsens due to the slower SpMV computations. ](assets/sig22 P-PIC optimized-20221205141100-fepixgd.pdf?p=8)

### Result

#### 能量守恒

​![image](assets/image-20221207143923-at3m477.png)​

#### 光滑表面

#### ![image](assets/image-20221207144221-8oukper.png)​

## Stage 3

> [第三遍，精读，这只有在报告论文或评审论文时需要进一步阅读的。你需要脱稿能描述该论文的问题、方法，以便你能报告或写review意见。如果需要实现该论文，还需再精读，了解其所有细节。通过实现论文能...](siyuan://blocks/20221205135757-zub029b)

‍

# 总结

结合了几何上的Power Diagram，处理了MPM中的权重问题。

计算效率？

‍

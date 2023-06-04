---
title: A Contact Proxy Splitting Method for Lagrangian Solid-Fluid Coupling
date: 2023-05-29
share: true
---

总结：本文提出了一种用于FEM可变形体与SPH流体之间的强流固耦合算法。首先，本文将SPH流体的运动、不可压条件等求解转化为能量函数优化问题。其次，本文将流固耦合问题转化为流体粒子与固体网格之间的碰撞问题，通过引入了IPC模型，在运动求解的同时处理了碰撞与耦合，并提出了强流固耦合对应的能量函数优化问题。最后，本文针对提出的优化问题使用了交替迭代方法，借助Contact Proxy实现了快速且稳定的求解。

- https://doi.org/10.48550/arXiv.2301.01976
- SIGGRAPH 2023

# A Contact Proxy Splitting Method for Lagrangian Solid-Fluid Coupling

> [!NOTE] Abstract
> We present a robust and efficient method for simulating Lagrangian solidfluid coupling based on a new operator splitting strategy. We use variational formulations to approximate fluid properties and solid-fluid interactions, and introduce a unified two-way coupling formulation for SPH fluids and FEM solids using interior point barrier-based frictional contact. We split the resulting optimization problem into a fluid phase and a solid-coupling phase using a novel time-splitting approach with augmented contact proxies, and propose efficient custom linear solvers. Our technique accounts for fluids interaction with nonlinear hyperelastic objects of different geometries and codimensions, while maintaining an algorithmically guaranteed non-penetrating criterion. Comprehensive benchmarks and experiments demonstrate the efficacy of our method.

- FEM 固体 + SPH 流体强耦合： 全拉格朗日视角
- SPH 的不可压缩、粘性转化为 Minimize Potential
- IPC模型建模流固边界（碰撞、摩擦）
- 计算加速、不稳定性上：Proxy Contact Energy，Matrix Free CG Solver

![[Pasted image 20230529140341.png]]

## 二次的不可压缩/粘性势能

NS:

$$
\rho _{s} \left( \frac{Dv_{s}}{Dt} \right) = \nabla \cdot \sigma + \rho_{s} \mathbf{g} + \mathbf{f_{ext}}
$$

此处的$\mathbf{f_{ext}}= \mathbf{f_{s \rightarrow s}}+\mathbf{f}_{f\rightarrow s}$

- FEM Solid → 本身使用能量函数建模。
- SPH → Incompressibility 通常是通过PPE求解得到，并非能量函数。

### Incompressibility Potential

$$
P_{I}(\mathbf{x})=\sum_{i} \frac{k_{I}}{2} V_{0}(J_{i} (\mathbf{x}) - 1)^{2} \quad J = \rho_{0} / \rho
$$

其中的：
$$
J_{i} ^{n+1} = J_{i}^{n}(1 + h \nabla \cdot \mathbf{v}_{i}^{n+1})
$$

主要是为了能够让其写成$\mathbf{x}$的二次型：

$$
J_{i}^{n} = \frac{\rho_{0}}{\sum_{j} m_{j} W_{ij}},\quad
\nabla \cdot \mathbf{ v}_{i} ^{n+1}=\sum_{j} \frac{m_{j}}{\rho_{j}^{n}} (\mathbf{v}_{j}^{n+1} - \mathbf{v}_{i}^{n+1})\cdot \nabla_{i} W_{ij}
$$

### Viscosity Potential

$$
P_{V}(\mathbf{x}) = \frac{1}{4} v \hat{h} \sum_{i,j}
\| \mathbf{v}_{ij}^{n+1}\|^{2}_{\mathbf{V}_{ij}}
$$
其中：
$$
\mathbf{V}_{ij} = 4(d+2) \frac{m_{i}m_{j}}{\rho_{i} + \rho_{j}} \frac{- \nabla_{i} W_{ij}(\mathbf{x}_{ij})^T}{\| \mathbf{x}_{ij}^{n} \|^{2} +0.01 \bar{h}^{2}}
$$

## Coupling

### Distance Function

$$
B_{sf}(\mathbf{x}_{s}, \mathbf{x}_{f}) = \sum_{q \in \mathcal{Q}_{f}} \sum_{e \in \mathcal{B}_{s}} s_{q} b(d^{PT}(\mathbf{x}_{q}, e), \hat{d})
$$

### Opt Form

$$
\Psi = \Psi_{s}, P = P_{I} + P_{V}, C_{sf}=B_{sf}+D_{sf}
$$

Implicit Euler:

$$
\begin{cases}
\mathbf{v}^{n+1} = \mathbf{v}^{n} + h M^{-1}(f_{ext} - \nabla P(\mathbf{x}^{n+1}) - \nabla \Psi(\mathbf{x}^{n+1}) - \nabla C(\mathbf{x}^{n+1})) \\
\mathbf{x}^{n+1} = \mathbf{x}^{n} + h v^{n+1}
\end{cases}
$$

## Efficient Solver

projected Newton method:

$$
\begin{bmatrix}
 \mathbf{H}_{f}  & G \\
G^{T} & \mathbf{H}_{s}
\end{bmatrix} = \begin{bmatrix}
\mathbf{g}_{f} \\
\mathbf{g}_{s}
\end{bmatrix}
$$

### Base Time Splitting

![[assets/Pasted image 20230529162338.png]]

- 优点：计算快
- 缺点：带来不稳定性

### Time Splitting with Contact Proxy

![[assets/Pasted image 20230529162832.png]]

> [!note]
> 此处要在FSI的求解中添加$\nabla \hat{C}_{sf}$。

$$
\hat{C}_{sf}=\text{Taylor Expand 2nd of } C_{sf}
$$

![[assets/Pasted image 20230529163011.png]]

![[assets/Pasted image 20230529163111.png]]

> [!warning] 
> 求解变量中没有$\mathbf{v}$

类似地，有：

![[assets/Pasted image 20230529163205.png]]


## Mindmap for SPH Boundary Handling

[[additionals/mindmap-for-sph]]
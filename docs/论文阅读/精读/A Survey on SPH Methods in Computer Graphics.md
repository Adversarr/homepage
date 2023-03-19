---
title: A Survey on SPH Methods in Computer Graphics
date: 2023-02-09T15:08:54Z
lastmod: 2023-02-25T11:10:02Z
share: true
tags:
  - SPH
  - IPC
  - PaperNotes
  - PhysicalSimulation
  - Survey
---


# A Survey on SPH Methods in Computer Graphics

#SPH​, #IPC​, #PaperNotes​, #PhysicalSimulation​, #Survey​

摘要：第三部分将介绍SPH流体模拟中引入粘性、涡量和表面张力的主要方法。粘性力主要描述了流体动能的耗散，主要有显式和隐式求解两类方法。借鉴了有限差分的显式方法，能够保持守恒量，并有较高的计算效率，是粘性力计算的主流方法。在涡量和表面张力的研究上，主要可以从宏观或微观上入手。引入涡量有助于降低SPH格式的数值耗散现象，能够更好保持流体的漩涡运动。表面张力描述了液体分子之间的相互吸引力，有助于更真实的流体表面的形成。

流体的空间离散的最基本方程为：


​![](assets/sphcgf20230210103947-20230210103949-i7fw8j1.png)​

在空间上的梯度，用SPH离散结果为：


​![](assets/sphcgf20230210103435-20230210103437-t31s7an.png)​


​![](assets/sphcgf20230210103651-20230210103653-569kb1l.png)​

## Materials

### Viscous Force

Definition and Derivation


​
![](assets/sphcgf-P12-20230209181405-20230209181406-1hsxwv9.png)​

Solver:

1. p12 - compute the Laplacian of the velocity field using an SPH formulation
2. p12 - use a strain rate based formulation

### Explicit Viscosity

对于SPH而言，标准的Laplacian公式为：


​![](assets/sphcgf-P12-20230209181735-20230209181736-vc5i82d.png)​

缺点：$W$的二阶导数变号甚至不连续

> sphcgf.pdf - p12 - the second derivative of the kernel changes sign inside the kernel domain and can even be discontinuous (e.g., cubic spline kernel

解决方法：

1. sphcgf.pdf - p12 - compute the second derivative by taking two first SPH derivatives

    > [Viscosity_Takahashi2015.cpp](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH/blob/master/SPlisHSPlasH/Viscosity/Viscosity_Takahashi2015.cpp)
    >

    $$
    \begin{aligned}
    \nabla v =\sum_j \frac{m_j}{\rho_j} \cdot (v_j - v_i) (\nabla w)^T\\
    \mathrm{visco~stress}_i = \rho_0 \mathrm{VisC} (\nabla v + \nabla v^T)
    \end{aligned}
    $$

2.  combine an SPH derivative with finite difference

    > [Viscosity_Standard.cpp](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH/blob/master/SPlisHSPlasH/Viscosity/Viscosity_Standard.cpp)
    >

    ​![](assets/sphcgf-P12-20230210102112-20230210102113-0ysvbg5.png)​

    其中h项仅用于消除nan

### Implicit Viscosity

四种方式：

#### Implicit strain rate based solver

原本是通过Stress Tensor来推导出的。

​![](assets/sphcgf-P12-20230211185136-20230211185137-q490gmb.png)​

#### Velocity Gradient decomposition

​![](assets/sphcgf-P13-20230211185213-20230211185215-ywolzda.png)​

#### Strain Rate Constraint Formulation

Strain Rate 指的是 Strain$E$对时间$t$的导数：

> ​![](assets/sphcgf-P12-20230209181405-20230209181406-1hsxwv9.png)​

用参数$\gamma$来降低Strain Rate：

$$
C_i(v) = (1 - \gamma) E_i
$$

借助拉格朗日乘子法，转化为求解如下的线性系统：

​![](assets/sphcgf-P13-20230213160916-20230213160918-bunoa7k.png)​

#### Implicit Laplacian Based Solver

​![](assets/sphcgf-P13-20230213161105-20230213161122-nnyy27l.png)​

## Vorticity

> 研究原因：Numerical Diffusion -- 类似于APIC

Turbulent details quickly get lost due to a coarse sampling of the velocity field or due to numerical diffusion]
### Vorticity Confinement

Foreach Particle:


​![](assets/sphcgf-P17-20230213194020-20230213194029-f7eoqzx.png)​

产生的力为：

$$
\begin{aligned}

F_i ^{vorticity} = \epsilon^V\left( \frac{\eta}{\|\eta\|} \times \omega_i\right)
\\
\eta = \sum_j \frac{m_j}{\rho_j} \|\omega _ j\| \nabla W_{ij}
\end{aligned}
$$

其中的 $\epsilon$ 是用户定义的一个参数。

‍

## Surface Tension

表面张力计算，只有4种方法。

1. surface tension minimizes the surface area which causes droplets of water to form a sphere when external forces are excluded
2. Cohesion and adhesion are important effects when simulating surface tension

### Microscopic Approach -- Micro Scope(ic) - 微观

逻辑上很简单：

$$
m \frac{Dv_i} {Dt}=\alpha \sum_{j} m_j(x_{j} - x_i)W_{ij}
$$

直接从原理上计算，即分子间吸引力。

计算[SurfaceTension_Becker2007.cpp](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH/blob/master/SPlisHSPlasH/SurfaceTension/SurfaceTension_Becker2007.cpp "SurfaceTension_Becker2007.cpp")

```cpp
const Vector3r &xi = m_model->getPosition(i);
Vector3r &ai = m_model->getAcceleration(i);
forall_fluid_neighbors_in_same_phase(
  const Vector3r xixj = xi - xj;
  const Real r2 = xixj.dot(xixj);
  if (r2 > diameter2)
   ai -= k / m_model->getMass(i) * m_model->getMass(neighborIndex) * (xi - xj) * sim->W(xi - xj);
  else
    ai -= k / m_model->getMass(i) * m_model->getMass(neighborIndex) * (xi - xj) * sim->W(Vector3r(diameter, 0.0, 0.0));
);
```

​![](assets/SPH_Tutorial-P25-20230214102524-20230214102526-nph09cr.png)​

### Macroscopic Approach

#### Cohension

> SPH_Tutorial.pdf - p26 - Instead of only considering cohesive forces, also forces to minimize the surface area are determined.

Compute as:

​![](assets/SPH_Tutorial-P26-20230214144749-20230214144751-qhwbwwy.png)​

另一方面，用最小化表面积为目标，需要计算液体表面的法向来计算Curvature（曲率）：

​![](assets/SPH_Tutorial-P26-20230214145602-20230214145603-tatkhca.png)​

​![](assets/SPH_Tutorial-P26-20230214145616-20230214145617-v8d8dte.png)​

#### Adhesions

模拟流体粒子之间的吸引力，和之前的公式几乎相同。

​![](assets/SPH_Tutorial-P26-20230214145722-20230214145723-djw84y5.png)​

W的计算可以参考式 P27

[SurfaceTension_Akinci2013.cpp](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH/blob/master/SPlisHSPlasH/SurfaceTension/SurfaceTension_Akinci2013.cpp "SurfaceTension_Akinci2013.cpp")

​![image](assets/image-20230214150101-t8jgww2.png)​

## Vorticity Revisit

在这篇文章中更详细地介绍了涡的形成和计算。

​![](assets/SPH_Tutorial-P27-20230214150742-20230214150744-pbtgb7w.png)​

> In Wikipedia: [Vortex](https://en.wikipedia.org/wiki/Vortex)
>
> ​![image](assets/image-20230214151008-7nbhv8s.png)​

### Vorticity Confinement

[Vorticity Confinement](https://en.wikipedia.org/wiki/Vorticity_confinement)

1. 不保存每一个粒子处的涡量，而是每一次都计算得到，
2. 计算涡量带来的力大小，通过力矩推导，需要确定一个系数
3. Velocity Field Smooth

​![](assets/SPH_Tutorial-P27-20230214151129-20230214152200-21bkyj0.png)​

### MicroPolar Model

#### Balance Law for Angular Momentum Conservation

> 参照Linear Momentum：
>

> ​![](assets/SPH_Tutorial-P8-20230214170846-20230214170848-8r3bj66.png)
>
> 其中用NS-Eq代入
>

> ​![](assets/SPH_Tutorial-P8-20230214175905-20230214175907-lobfrik.png)
>

> ​![](assets/SPH_Tutorial-P8-20230214180033-20230214180034-r9uejqw.png)​​

角动量相同的有如下的公式，其中的$\Theta$表示一个标量：


​![](assets/SPH_Tutorial-P28-20230214170055-20230214170056-n7sotpg.png)  ​

其中的$T$可以计算为：


​![](assets/SPH_Tutorial-P28-20230214175456-20230214175458-5rz5r2k.png)

从而速度更新计算公式可以写作


​![](assets/SPH_Tutorial-P28-20230214175800-20230214175801-0e55uc6.png)​

> Algorithm:
>

> ​![](assets/SPH_Tutorial-P28-20230214180132-20230214180133-g1kv033.png)​
>
> Compare to standard solver
>

> ​![](assets/SPH_Tutorial-P9-20230214181032-20230214181042-mcvg0mr.png)​
>
> ​![](assets/SPH_Tutorial-P18-20230214181246-20230214181306-a12et4o.png)​

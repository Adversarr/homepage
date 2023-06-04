---
share: true
---

烟雾的模拟是流体模拟中的一个重要对象。相较于普通流体，其具有额外的烟雾浓度、温度参数。烟雾模拟将物理公式简化为线性公式，烟雾浓度和温度对速度的影响合为浮力，作为外力项进行处理。同时，由于温度变化导致流体密度产生变化，压强求解有相应的修正。


## 烟雾模拟

多两个变量：

1. 烟雾浓度$s$
2. 温度 $T$

> 需要模拟整个环境中的空气，而非烟雾本身。即使是浓度几乎为0的空气，也会对于整个烟的速度、密度场有影响。

对于非源点，要求随体导数为0：

$$
\frac{DT}{Dt}= 0\qquad \frac{Ds}{Dt}= 0
$$
考虑源的情况下：

![[assets/Pasted image 20230417165111.png]]

$r$ 的大小控制烟雾的产生速度。

离散形式为

![[assets/Pasted image 20230417165156.png]]


此外，浓度和温度都用人为的扩散模型进行迭代，即

$$
\frac{DT}{Dt}= k_{T} \nabla \cdot \nabla T
$$

上面的扩散模型需要CFL条件来保证不发散：

$$
\Delta t \le \sim \frac{\Delta x^{2}}{6k_{T}}
$$

换一种非物理的方式，可以采用 **heat kernel** 卷积当前的物理场来模拟扩散。

或者使用heat kernel的隐式形式：

![[assets/Pasted image 20230417191615.png]]

## 浮力

$T$ 和 $s$ 都对于速度产生影响，额外施加加速度来模拟：

$$
\text{Bouyancy: }\vec b =\left(\alpha s - \beta(T - T_{amb}) \right) \vec g
$$

- 空气：$s=0, T=T_{amb}$ ，则$\vec b = 0$
- 特别热：$T \gg T_{amb}$ ，$\vec b = - k \vec g$
- 浓度高：下沉，故有 $\vec b = k \vec g$

## Variable Density Solves（处理密度变化）

$$
\rho \approx \rho_{0}\left(1 + \alpha s - \frac{1}{T_{amb}}(T- T_{amb})\right)
$$

从而，动量更新：

![[assets/Pasted image 20230417200742.png]]

如果$\alpha s - \beta \Delta t\ll 1$ 且有 $\Delta p'=\Delta p +\rho_{0}\vec g$，那么

![[assets/Pasted image 20230417200859.png]]

## 密度控制

不强求控制，只要求不爆炸。

可以采用一个控制场来调节需要的Divergence，来“模拟”密度变化带来的影响。
---
share: true
---

# General ADMM

> Note: [admm.pdf (cmu.edu)](https://www.stat.cmu.edu/~ryantibs/convexopt/lectures/admm.pdf)

consider problem:

$$
\min f(x)+g(z)\quad s.t. Ax+Bz =c
$$

ADMM's dual problem is:

$$
\min L_{\rho}(x,z,u)=f(x)+g(z)+u^{T}(Ax+Bz-c)+\rho/2 \|Ax+Bz-c\|_{2}^{2}
$$

ADMM update becomes:

$$
\begin{aligned}
x^{(k)} = \arg\min_{x} L_{\rho}(x, z^{(k-1)}, u^{(k-1)})\\
z^{(k)} = \arg\min_{x} L_{\rho}(x^{(k)}, z, u^{(k-1)})\\
u^{(k)} = u^{(k-1)} + \rho(Ax+Bz-c)
\end{aligned}
$$

Or use `General consensus ADMM`:

![[assets/Pasted image 20230401110338.png]]

# ADMM Solver for implicit time integrators.

Implicit time integration says:

$$
\begin{cases}
x^{k+1} = x^{k} + h v^{k+1} \\
v^{k+1} = v^{k} + h\frac{1}{m}(f(x_{k+1}))
\end{cases}
$$

Consider:

$$
f = f_{i}+f_{e}
$$

and $f_{i}$ does not change during the timestep, Then, following equation is to be solved.

$$
x = x^{k}+hv^{k} + h^{2} \frac{1}{m}f(x)\implies (x-y) - h^{2}\frac{1}{m}f_{i}(x)=0
$$
where:
$$
y = x^{k}+hv^{k}+h^{2}f_{i}(x^{k}) /m
$$

## Assumptions 

In the simulation, we only consider `Conservative` forces, i.e.

$$
\exists U(x),\text{ such that } f_{i}(x)=-\nabla U(x)
$$

Every non-conservative force are seen as `External`. such as incompressibility.

Then, we have: (if $U$ is convex)

$$
(x-y) + h^{2}\frac{1}{m}f_{i}(x)=0\Leftarrow x=\arg\min_{x}\frac{1}{2}\|x-y\|_{M}^{2} + U(x)
$$

Let $E(x) = \mathrm{Rhs} = I(x) + U(x)$, one critical observation is that, because we are finding some point that can BALANCE these forces, **we does not need the optimal, we only need the saddle point.**

That is to say, even ADMM may not guarantee the convergence to the optimal, but we can also stop at any point, where the gradient is small enough.

## Contact

### Description

Because now we focus on `Fluid-Rigid-Couping`, we only need to consider the `V-T` collision.

Given the normal $n_{t}$ of triangle $t$, we can estimate the distance as:

$$
|(v-v_{0})\cdot n_t|
$$

If we want to eliminate the collision using a Potential Function

- Force to $v$ should be applied parallel to $n_t$

The main problem of this method is, the stiffness of the potential is HARD to decide, and stiffness may become INFINITELY LARGE, making the problem extremely hard to solve.

## Coding Notations

When coding, we say:

- $y$ is the `inertia position`, because it describes the FIRST law of Newton
- $f$ describes the `constraint`
- $u$ is the ADMM multiplier, describng the difference between $Dx$ and $z$
- $z$ is the slacking variable, describing the input for each constraint. e.g. for spring, `x - y`, for fem, `DeformationGradient`
- $c$ describes Incremental Contact Potential related stuff.

we have two strategies, one is traditional, and another is consensus.

### Example: mass spring

For spring $i$, the potential is computed using the diff in position:

$$
z_{i}= x_{il}- x_{ir}
$$

and the potential is defined as:


$$
U_{i}(z)=(\|z\|-l_{i})^{2}
$$

$u$ describes the difference between $Dx$ and $z$

To solve $z$ parallelly, we can apply CG, or QN method.

### The choice of $W$

$W$ is chosen to be identity scaled by some given scalar.

For spring model:

$$
W_{i}= \sqrt{k_{i}} I
$$

consider original problem:

$$
\min I(x) + U(Dx)
$$

ADMM Rewrites as:

$$
\min I(x) + U(z) + \frac{1}{2}\|Dx-z+u\|_{W}^2
$$

with $z, u$ fixed:

$$
x=\arg\min_{x} I(x) + \frac{1}{2}\|Dx-z+u\|_{W}^2
$$

actually $W$ measures the balance between $z-u$ and $y$.

for $StVK$ and other FEM model, in paper `Quasi-Newton Methods for Real time Simulation of HyperElastic Models`, author proposed that 

$$
\lambda+2\mu
$$

is a good choice.

## HyperElastic Model

for Hyper elastic model, the slack variable is computed as the slack of $F$ instead of $Dx$

$$
U(z)+\|Dx -Rz+u\|_{W}^{2}
$$

## Contact model

### Incremental Potential contact

碰撞/相交是指，对于任意两个单纯复形，其中的两个不同单纯形之间的距离为$0$。将所有0-单纯形的位置视为自变量，距离为0实际上描述了一个高维空间中的流形 $d(\mathbf x) = 0$。

考虑一维空间$\mathbb R$中的两个小球，其位置坐标分别为$x, y$，时间积分中的优化变量为$\mathbf x= (x, y)\in \mathbb R^{2}$。则两个小球碰撞是指 $d(\mathbf x) = x-y = 0$，如下图所示。

![[assets/Pasted image 20230407140753.png]]

假定当前时间步的位置更新为$\mathbf x_{n} \rightarrow \mathbf x_{n+1}$，由隐式时间积分公式可知，我们认为在时间当前时间步内，所有的单纯形按照匀速直线运动，如下图所示。由于其中穿越了线性流形 $x-y = 0$，即存在时刻 $t \in [0, 1]$，使得$d(t \mathbf x_{n+1} + (1 - t) \mathbf x_{n}) = 0$ ，因此在这一时间步中发生了碰撞，需要进行正确的碰撞处理。

![[assets/Pasted image 20230407140709.png]]

基于惩罚的IPC算法在原先隐式时间积分公式的基础上，加入了碰撞项 $B(\mathbf x)$

$$
B(\mathbf x) = B(d(\mathbf x)) = \begin{cases}
-(d - \hat d) ^ {2} \log \left(\frac{d}{\hat d}\right) & 0 < d < \hat d\\
0 & \mathrm{otherwise}
\end{cases}
$$
优化问题为：

$$
\arg\min_{\mathbf{x}} E(\mathbf{x}) + B(\mathbf{x})
$$

碰撞的处理使用内点法规避，内点法要求每一次时间步更新 $\mathbf{x}_{n} \rightarrow \mathbf{x}_{n+1}$ 都满足

$$
\forall t \in [0, 1], E((1-t)\mathbf{x}_{n} + t \mathbf{x}_{n+1}) + B((1-t)\mathbf{x}_{n} + t \mathbf{x}_{n+1}) < \infty
$$

IPC通过内点法完全规避了所有的可能碰撞。

### 耦合 IPC

原先的IPC公式在每一个迭代步的动力学解算过程都需要进行连续碰撞检测。其对于 $F=E+B$ 进行带SPD预处理的牛顿迭代法进行带碰撞的动力学解算：

1. 计算可能出现碰撞的所有单纯形对，用于计算$B$的梯度和Hessian矩阵；
2. 计算 $E$ 的梯度和Hessian矩阵；
3. 计算牛顿下降方向；
4. 对该方向执行带碰撞检测的线搜索，确定当前迭代步的位置更新；

由于最后一步的线搜索执行了碰撞检测，并且该方向是一个下降方向，步长保证无碰撞发生，因此IPC确保了时间积分能够正确收敛到能量函数最小的位置，并且该时间步不发生穿透。

该算法的缺点在于，即使 $E$ 具有一个固定的 Hessian 矩阵，例如弹簧质点模型，由于 $\mathrm{Hessian} B$ 在每一个时间步都有可能发生变化，其也无法应用稀疏 Cholesky 分解等预计算加速方法进行快速矩阵计算。这对于复杂场景、复杂对象的解算是十分不利的。

# ADMM-IPC 解耦合

我们对于

$$
F=E(\mathbf{x})+B(\mathbf{x})
$$

也应用 ADMM进行计算：

$$
\arg\min _{\mathbf{x}} F(\mathbf{x})\Rightarrow \arg \min_{x} E(\mathbf{x}) + B(\mathbf{d}) \quad s.t.D \mathbf{x}=\mathbf{d}
$$

在流体与网格物体耦合的过程中，主要考虑的是点-三角类型的碰撞处理。例如对于流体粒子 $\mathbf{f}\in \mathbb{R}^{3}$ 和三角网格点$\mathbf{x}_{0}, \mathbf{x}_{1}, \mathbf{x}_{2}$，流体粒子到三角面的法向的距离通过下式给出。

$$
\left|(\mathbf{f} - \mathbf{x}_{0})\cdot \frac{(\mathbf{x}_{1} - \mathbf{x}_{0})\times (\mathbf{x}_{2} - \mathbf{x}_{0})}{\|(\mathbf{x}_{1} - \mathbf{x}_{0})\times (\mathbf{x}_{2} - \mathbf{x}_{0})\|}\right|
$$

则

$$
\mathbf{d} = [\mathbf{f} ; \mathbf{x}_{0}; \mathbf{x}_{1}; \mathbf{x}_{2}]
$$

ADMM给出如下形式的更新策略：

$$
\begin{aligned}
\mathbf{x}^{(k)} &= \arg\min_{\mathbf{x}} F(\mathbf{x}, \mathbf{d}^{(k-1)}, u^{(k-1)}) = E(\mathbf{x}) +  \frac{1}{2}\| D\mathbf{x} - \mathbf{d}\|_{W}^{2}\\
\mathbf{d}^{(k)}& = \arg\min_{\mathbf{d}} F(\mathbf{x}^{(k)}, \mathbf{d}, u^{(k-1)})\\
u^{(k)} &= u^{(k-1)} + (D \mathbf{x} - \mathbf{d})
\end{aligned}
$$

如果对于 $E$ 的解算也使用ADMM方法进行，则整个计算流程可以被简化为：

$$
\begin{aligned}
\mathbf{x}^{(k)} &=\arg\min_{\mathbf{x}} I(\mathbf{x}) + \frac{1}{2} \| D_{z} \mathbf{x} - \mathbf{z}^{(k-1)} +\mathbf{u}_{z}^{(k-1)}\|_{W}^{2} +  \frac{\rho}{2} \| D_{d} \mathbf{x} - \mathbf{d}^{(k-1)} +\mathbf{u}_{d}^{(k-1)}\|_{2}^{2}\\
\mathbf{z}^{(k)} &= \arg\min_{\mathbf{z}} U(\mathbf{z}) + \frac{1}{2} \| D_{z} \mathbf{x}^{(k)} - \mathbf{z} + \mathbf{u}_{z}^{(k-1)}\|_{W}^{2}\\
\mathbf{u}_{z}^{(k)} &=\mathbf{u}_{z}^{(k-1)}+ (D_{z} \mathbf{x}^{(k)} - \mathbf{z}^{(k)})\\
\mathbf{d}^{(k)}&=\arg \min_{\mathbf{d}} B(\mathbf{d}) + \frac{\rho}{2} \| D_{d} \mathbf{x}^{(k)} - \mathbf{d} + \mathbf{u}_{d}\|_{2}^{2}\\
\mathbf{u}_{d}^{(k)}&=\mathbf{u}_{d}^{(k-1)} +\rho (D_{d} \mathbf{x}^{(k)} - \mathbf{d}^{(k)}) \qquad (\text{always} =0)
\end{aligned}
$$

对于上述迭代，因为 $I(\mathbf{x})$ 为二次型，第一步可以使用Cholesky LLT分解进行加速求解。（Global Step）

对于第2、4步，每一个单独的 $\mathbf{z}, \mathbf{d}$ 都可以进行并行计算。（Local Step）

对于第3、5步，仅仅是简单的稀疏矩阵乘法和向量加法，可根据需要进行并行化处理。

需要注意的是，对于第1步，需要使用碰撞检测确定最大可行步长，即：令

$$
\tilde x^{(k)}=\arg\min_{\mathbf{x}} I(\mathbf{x}) + \frac{1}{2} \| D_{z} \mathbf{x} - \mathbf{z}^{(k-1)} +\mathbf{u}_{z}^{(k-1)}\|_{W}^{2} +  \frac{1}{2} \| D_{d} \mathbf{x} - \mathbf{d}^{(k-1)} +\mathbf{u}_{d}^{(k-1)}\|_{2}^{2}
$$

以及 $t_{\min}=CCD(x, \hat x)$ 表示最近的碰撞时刻，则位置更新为：

$$
x^{(k)} = t_{\min}\tilde x^{(k)} + (1-t_{\min})x^{(k - 1)}
$$
该算法利用每一步$x^{(k)}\rightarrow x^{(k+1)}$ 无碰撞来保证整个时间步内不发生非物理的穿透。而这一保证直接给出了 $\mathbf{u}_{d} = 0$

## 碰撞解算

### 问题简化

对于ADMM-IPC，碰撞解算就是 $\mathbf{d}$ 的解算。根据前文的分析，碰撞可能出现在任何出现位置更新的时刻，图示如下：


![[assets/Pasted image 20230407140003.png]]

在从$x_n$到$x_{n+1}$更新过程中，由于动力学解算的最小化问题中并不考虑碰撞，则可能穿越不可行区域，即发生了碰撞。

如图，假设 $\hat x = \arg\min_{\mathbf{x}} I(x) + \frac{1}{2}\|D_{z} \mathbf{x}-\mathbf{z} - \mathbf{u}_{z}\|_{W}^2$，从$\mathbf{x}$ 到$\hat{\mathbf{x}}$ 存在碰撞：

![[assets/Pasted image 20230407170920.png]]

计算$D^{-1}\mathbf{d}$ 应当有如下的位置：

![[assets/Pasted image 20230407201336.png]]


IPC算法指出，函数 $B$ 中的权重系数是可以是任意大的，其仅仅影响碰撞解算与动力学解算中对于碰撞的权衡，而不影响其迭代的最终结果。因此，我们可以选取在不越过接触边界的任一位置来计算$\hat{\mathbf x}$。（如下图所示）

![[assets/Pasted image 20230407201345.png]]

## 碰撞计算

对于距离函数$|x- y|$进行可视化可知

![[assets/Pasted image 20230407163350.png]]

对于 $B(\mathbf{d}) = B(|x-y|)$ 的梯度场可视化如下（选取$\hat d = 0.5$）：

![[assets/Pasted image 20230407164511.png]]

一个重要的观察是，由于 $B$ 是 $d$ 的函数，而 $d$ 是 $\mathbf{d}$ 的函数，由链式法则可知：

$$
\nabla_{\mathbf{d}} B(\mathbf{d})=\frac{\mathrm{d} B}{\mathrm{d} d} \nabla_{\mathbf{d}} d
$$

由于$B$的权重可以任意的大，因此优化问题可以被转化为：

$$
\mathbf{d}= \arg\min_{B(\mathbf{d}) = 0} \frac{1}{2} \| D \mathbf{x}^{(k)} - \mathbf{d}\|
$$

将该公式直接合并到 $\mathbf{x}$ 的求解中可以得到：

$$
\begin{aligned}
\mathbf{n} &= \mathrm{normalize}(\nabla b)\\
L &= (\hat {\mathbf{x}}- \mathbf{x})\cdot \mathbf{n}\\
D^{-1}\mathbf{d}&= \left(t_{I} L+\hat d\right) \mathbf{n} + \hat{\mathbf{x}}
\end{aligned}
$$
### ADMM-IPC

$$
\begin{aligned}
&\textbf{Algorithm } \text{ADMM-IPC}\\
&\textbf{Input: } \text{Position of last time step } \mathbf{x}\\
& \mathbf{x}^{(1)} = \mathbf{x}, \mathbf{u}_{z}^{(0)} = \mathbf{0}, k=1\\
& \textbf{While } \text{not converged} \textbf{ do:}\\
& \quad \mathbf{z}^{(k)}\leftarrow\text{DynamicLocalStep()}\\
& \quad \hat{\mathbf{x}}^{(k)}\leftarrow \text{DynamicGlobalStep()}\\
&\quad \mathbf{u}_{z} ^{(k)} \leftarrow \mathbf{u}_{z}^{(k-1)} + (D_{z} \mathbf{x} ^{(k)}- \mathbf{z}^{(k)})\\
& \quad \textbf{For } \text{i in range c} \textbf{ do:}\\
& \quad \quad \mathbf{d}^{(k)} \leftarrow \text{ContactLocalStep}(\mathbf{x}^{(k)}, \hat{\mathbf{x}}^{(k)})\\

&\quad \quad \hat{\mathbf{x}}^{(k)} \leftarrow \text{ContactGlobalStep}(\mathbf{d}^{(k)}, \hat{\mathbf{x}}^{(k)})\\
& \quad \mathbf{x}^{(k+1)} \leftarrow (1-t_{I}) \mathbf{x}^{(k)} +t_{I} \hat{\mathbf{x}}^{(k)}\\
&\quad k \leftarrow k + 1\\
& \mathbf{x}^{*} \leftarrow \mathbf{x}^{(k)}
\end{aligned}
$$

## Dynamic Solver


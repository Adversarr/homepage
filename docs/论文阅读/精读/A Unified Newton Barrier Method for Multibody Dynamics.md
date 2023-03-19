---
title: A Unified Newton Barrier Method for Multibody Dynamics
date: 2022-12-05T11:18:40Z
lastmod: 2022-12-05T13:37:17Z
share: true
tags:
  - IPC
  - PhysicalSimulation
  - PaperNotes
---


# A Unified Newton Barrier Method for Multibody Dynamics

 #IPC ​, #PhysicalSimulation ​, #PaperNotes ​

---

## A Unified Newton Barrier Method for Multibody Dynamics

1. 多体动力学的
2. 统一、可控的
3. 积分框架

设计了：

1. 一般约束的换元策略
2. 无符号距离的非线性关节约束的统一公式
3. 不等式约束的、高阶时间积分、Rayleigh阻尼的通用恢复模型

Background：

1. IPC -- 将碰撞添加到目标函数内
2. 相关进展：CIPC（多维度ipc）
3. Medial IPC：中轴骨骼

#### 线性等式约束

$$
c_i ' x = s_i
$$

应用：

1. 点连接
2. 铰链$p_1 = p_2, \quad p_1' = p_2'$
3. 平移：点仅能在一个固定面内移动

#### 非线性约束

转化为附加势能：

$$
P_{NEq} (x) = \frac 1 2 k (c(x))^2,\quad k \gg 1
$$

1. 相对滑动
2. 距离约束

#### 不等式约束

滑动范围、旋转范围：使用障碍项（类似于IPC）

### 能量恢复

使用半隐式Rayleigh阻尼力。

### 优缺点

1. 统一的求解器
2. 小场景的性能远远低于已有的求解器
3. 无GPU加速

> 隐式时间积分器一定有耗散。

## Affine Body Dynamics

Rigid-IPC：

1. 在刚体仿真过程中添加了基于ipc的势能
2. 线性细分

### 本文思路

刚体-6dof

使用仿射变换：  
$x_k = A_b(t) \bar x_k + p _ b (t)= J(\bar x_k) q$

1. 动能：$\frac 1 2 \dot q ' \left(\int \rho J(x)'J(x) \mathrm d\Omega\right)q$
2. 运动方程：$M\ddot q = - \nabla V(q) + f$ 其中$f$为外力

Orthogonality Potential：

保持势能：

$$
V = \kappa\nu \| AA' - I_3\|_F^2
$$

Affine IPC：

$$
E(q) = V_C(q) + V_F(q) + \sum _ {b} E_b ( q_b )
$$

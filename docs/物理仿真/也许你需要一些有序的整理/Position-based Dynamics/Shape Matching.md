---
title: Shape Matching
date: 2023-03-06T12:42:15Z
lastmod: 2023-03-06T13:06:31Z
share: true
tags:
  - Deformables
  - ShapeMatching
---


# Shape Matching

Keys: #Deformables​, #ShapeMatching​,

## What is shape matching and Why

Pbd: mainly useful for thin shells.

For Volumetric -- less ideal.Proper solution:

* FEM
* Slow and complicated
* and have stability issues

Shape matching:

* Very useful, e.g. image registration.

### Mesh-less methods

* No springs, no edges, just points.
* No explicit connectivity
* Not just physics based.

Rest pose → Current pose.

Find an optimal Translation and Rotation match the "→"

​![image](../../其他教程/assets/image-20230306130118-8058oi4.png)

Force ⇒ *Attract* points to the "goal" positions.

## Procrustes problem

Find $t_0, t, R$ to minimize:

$$
\sum_ i w_i (R(x_i^0 - t_0) + t - x_i)^2
$$

Then, time-stepping:

$$
g_i = R(x_i ^0 - x_{cm}^0) + x_{cm}\\
v_i(t+h) = v_i(t) + \alpha \frac{g_i(t) - x_i(t)}{h} + h f_{ext}(t) / m_i\\
x_i(t+h) = x_i(t) + h v_i(t+h)
$$

## References

1. [zhuanlan.zhihu.com/p/459...](https://zhuanlan.zhihu.com/p/459933370)
2. [www.youtube.com/watch?v=...](https://www.youtube.com/watch?v=6Zsv2PyeQ5c&list=PL_a9tY9IhJuM2dIVCH_ZC0Pn5871eDY7_&index=3)

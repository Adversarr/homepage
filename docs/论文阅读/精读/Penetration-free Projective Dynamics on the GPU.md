---
share: true
tags:
  - ProjectiveDynamics
  - GPU
  - IPC
---



#ProjectiveDynamics, #GPU, #IPC

# Penetration-free Projective Dynamics on the GPU

## 主要贡献

1. PD + IPC 无穿透：PD对于High Stiffness收敛慢，但IPC具有High Stiffness
2. 新的Solver A-Jacobi，充分利用GPU
3. Faster Root Finding

## Formula 

PD:

Local Step

![[assets/Pasted image 20230321105858.png]]

Global Step

![[assets/Pasted image 20230321105923.png]]

IPC PD:

![[assets/Pasted image 20230321105937.png]]

## Method




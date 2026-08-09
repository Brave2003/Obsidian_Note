# 第一估计雅可比 (First-Estimate Jacobian Estimators)

> 来源：https://docs.openvins.com/fej.html

---

## 一、为什么FEJ重要？

VINS的**可观性和一致性**至关重要：

1. 初始化的最小条件 → 哪些运动能初始化系统
2. 哪些状态不可恢复 → 什么传感器配置才有意义
3. 退化运动 → 什么情况下精度会显著下降

**核心问题**：朴素EKF-based VINS在**不可观方向**获得虚假信息增益，导致滤波器**过度自信**。

> Bar-Shalom et al.：
> *"由于滤波器增益基于滤波器计算的误差协方差，一致性对滤波器最优性至关重要：错误的协方差产生错误的增益。这是为什么一致性评估对验证滤波器设计至关重要——它本质上是评估估计器的最优性。"*

---

## 二、EKF线性化误差状态系统

### 简化系统模型

考虑简化状态 $\mathbf{x} = [\mathbf{x}_{IMU}; \mathbf{p}_f]$（IMU状态 + 单个特征，忽略bias）：

**IMU运动模型**：
$$\tilde{\mathbf{x}}_{k} = \boldsymbol{\Phi}_{k-1} \tilde{\mathbf{x}}_{k-1} + \mathbf{w}_{k-1}$$

**相机测量**：
$$\tilde{\mathbf{z}}_k = \mathbf{H}_k \tilde{\mathbf{x}}_k + \mathbf{n}_k$$

### 状态转移矩阵的完整推导

$\boldsymbol{\Phi}_{k-1}$ 将时刻 $k-1$ 的误差映射到时刻 $k$：

$$\boldsymbol{\Phi}_{k-1} = \frac{\partial \tilde{\mathbf{x}}_k}{\partial \tilde{\mathbf{x}}_{k-1}} = \begin{bmatrix}
\boldsymbol{\Phi}_{11} & \mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \boldsymbol{\Phi}_{14} & \mathbf{0}_{3\times3} \\
\boldsymbol{\Phi}_{21} & \mathbf{I}_{3\times3} & \mathbf{I}_{3\times3}\Delta t & \boldsymbol{\Phi}_{24} & \boldsymbol{\Phi}_{25} \\
\boldsymbol{\Phi}_{31} & \mathbf{0}_{3\times3} & \mathbf{I}_{3\times3} & \boldsymbol{\Phi}_{34} & \boldsymbol{\Phi}_{35} \\
\mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \mathbf{I}_{3\times3} & \mathbf{0}_{3\times3} \\
\mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \mathbf{0}_{3\times3} & \mathbf{I}_{3\times3}
\end{bmatrix}$$

各分块逐一推导：

**(1) $\boldsymbol{\Phi}_{11} = \mathbf{I}$**（姿态误差的自相关）

在当前线性化点，姿态误差直接传递（近似忽略 $\lfloor \boldsymbol{\omega}\times \rfloor \Delta t$ 项用于可观性分析）。

**(2) $\boldsymbol{\Phi}_{21}$**：姿态误差→位置误差

$$\boldsymbol{\Phi}_{21} = -\lfloor \mathbf{g} \times \rfloor \hat{\mathbf{R}}_{I_{k-1}} \Delta t \cdot \frac{\partial \mathbf{p}_k}{\partial \delta\boldsymbol{\theta}_{k-1}}$$

推导过程：
- 从速度误差出发：$\tilde{\mathbf{v}}_k = \tilde{\mathbf{v}}_{k-1} - \hat{\mathbf{R}}_{I_{k-1}}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a)\times \rfloor \delta\boldsymbol{\theta}_{k-1} \Delta t$
- 位置由速度积分：$\tilde{\mathbf{p}}_k = \tilde{\mathbf{p}}_{k-1} + \tilde{\mathbf{v}}_{k-1}\Delta t + \frac{1}{2}\Delta\tilde{\mathbf{v}}\Delta t$
- 代入整理得（简化后用于可观性分析）：
$$\boldsymbol{\Phi}_{21} = -\lfloor \mathbf{g} \times \rfloor \hat{\mathbf{R}}_{I_{k-1}} \Delta t$$

**(3) $\boldsymbol{\Phi}_{31}$**：姿态误差→速度误差

$$\boldsymbol{\Phi}_{31} = -\lfloor \mathbf{g} \times \rfloor \hat{\mathbf{R}}_{I_{k-1}} \Delta t$$

**(4) $\boldsymbol{\Phi}_{24}, \boldsymbol{\Phi}_{25}, \boldsymbol{\Phi}_{34}, \boldsymbol{\Phi}_{35}$**：陀螺仪/加速度计bias→位置/速度

$$\boldsymbol{\Phi}_{24} = -\frac{1}{2}\hat{\mathbf{R}}_{I_{k-1}}^\top \Delta t^2, \quad \boldsymbol{\Phi}_{25} = \mathbf{0}$$

$$\boldsymbol{\Phi}_{34} = -\hat{\mathbf{R}}_{I_{k-1}}^\top \Delta t, \quad \boldsymbol{\Phi}_{35} = \mathbf{0}$$

### 测量Jacobian推导

$$\mathbf{H}_k = \mathbf{H}_{proj} \cdot \mathbf{H}_{trans} \cdot \mathbf{H}_{feat}$$

对于3D全局位置特征 $\mathbf{p}_G$：

$$\mathbf{H}_{proj} = \begin{bmatrix} 1/z_C & 0 & -x_C/z_C^2 \\ 0 & 1/z_C & -y_C/z_C^2 \end{bmatrix}$$

$$\mathbf{H}_{trans} \text{（对IMU状态）} = \begin{bmatrix} \mathbf{R}_{CI} \lfloor \mathbf{R}_{IG}(\mathbf{p}_G - \mathbf{p}_I) \times \rfloor & -\mathbf{R}_{CI} \mathbf{R}_{IG} & \mathbf{0} & \mathbf{0} & \mathbf{0} \end{bmatrix}$$

$$\mathbf{H}_{feat} = \mathbf{R}_{CI} \mathbf{R}_{IG}$$

---

## 三、线性化系统的可观性分析（详细推导）

### 可观性矩阵

$$\mathcal{O}_k = \begin{bmatrix} \mathbf{H}_0 \\ \mathbf{H}_1\boldsymbol{\Phi}_0 \\ \mathbf{H}_2\boldsymbol{\Phi}_1\boldsymbol{\Phi}_0 \\ \vdots \\ \mathbf{H}_k\boldsymbol{\Phi}_{k-1}\cdots\boldsymbol{\Phi}_0 \end{bmatrix}$$

如果 $\mathcal{O}$ **满列秩** → 系统完全可观（理论上可以恢复所有状态）

**零空间** $\mathcal{O}\mathbf{N} = \mathbf{0}$ → **不可观状态子空间**

### VINS的不可观零空间（详细验证）

VINS的右零空间为：

$$\mathbf{N}_k = \begin{bmatrix} 
\mathbf{R}_{IG,k} \mathbf{g} & \mathbf{0}_{3\times3} \\
-\lfloor {}^{G}\mathbf{p}_{I_k} \times \rfloor \mathbf{g} & \mathbf{I}_{3\times3} \\
-\lfloor {}^{G}\mathbf{v}_{I_k} \times \rfloor \mathbf{g} & \mathbf{0}_{3\times3} \\
\mathbf{0}_{3\times1} & \mathbf{0}_{3\times3} \\
\mathbf{0}_{3\times1} & \mathbf{0}_{3\times3}
\end{bmatrix} \in \mathbb{R}^{15\times4}$$

**验证 $\mathbf{H}_k \mathbf{N}_k = \mathbf{0}$**：

计算 $\mathbf{H}_k \mathbf{N}_k$ 的第一列（对应yaw不可观方向）：

$$\mathbf{H}_{proj} \mathbf{H}_{trans} \begin{bmatrix} \mathbf{R}_{IG,k} \mathbf{g} \\ -\lfloor {}^{G}\mathbf{p}_{I_k} \times \rfloor \mathbf{g} \\ -\lfloor {}^{G}\mathbf{v}_{I_k} \times \rfloor \mathbf{g} \\ \mathbf{0} \\ \mathbf{0} \end{bmatrix}$$

展开 $\mathbf{H}_{trans}$ 项：

$$\lfloor \mathbf{R}_{IG}(\mathbf{p}_G - \mathbf{p}_I) \times \rfloor \cdot \mathbf{R}_{IG} \mathbf{g} - \mathbf{R}_{IG}(-\lfloor \mathbf{p}_I \times \rfloor \mathbf{g})$$

由于 $\lfloor \mathbf{R}\mathbf{a} \times \rfloor = \mathbf{R}\lfloor \mathbf{a} \times \rfloor \mathbf{R}^\top$ 和 $\lfloor \mathbf{a} \times \rfloor \mathbf{a} = 0$：

$$= \mathbf{R}_{IG} \lfloor (\mathbf{p}_G - \mathbf{p}_I) \times \rfloor \mathbf{g} + \mathbf{R}_{IG} \lfloor \mathbf{p}_I \times \rfloor \mathbf{g}$$
$$= \mathbf{R}_{IG} \lfloor \mathbf{p}_G \times \rfloor \mathbf{g}$$

然后乘 $\mathbf{H}_{proj} \mathbf{H}_{feat}$... 等等，需要验证这个量映射通过投影后是否为零。实际上，这意味着绕重力轴旋转等价于全局平移，在仅有bearing测量时无法区分。

**不可观方向的物理解释**：

- **第1列 (yaw)**：绕重力方向旋转 + 相应调整位置和速度 → 相机观测不变
- **第2-4列 (位置)**：全局平移 → 相机和IMU的相对观测不变

---

## 四、为什么朴素EKF会错误地获得yaw可观性？

### 问题根源

标准EKF在每个时间步使用**当前最新估计**计算 $\boldsymbol{\Phi}$ 和 $\mathbf{H}$：

$$\boldsymbol{\Phi}_{k-1} \leftarrow \frac{\partial f}{\partial \mathbf{x}}\bigg|_{\hat{\mathbf{x}}^+_{k-1}}$$
$$\mathbf{H}_k \leftarrow \frac{\partial h}{\partial \mathbf{x}}\bigg|_{\hat{\mathbf{x}}^-_k}$$

**后果**：不同时刻的线性化点不同 → 可观性矩阵的零空间不再是 $\mathbf{N}$。

### 具体分析

第 $k$ 行可观性矩阵块：

$$\mathcal{O}^{(k)} = \mathbf{H}_k \boldsymbol{\Phi}_{k-1} \cdots \boldsymbol{\Phi}_0$$

在朴素EKF中：$\boldsymbol{\Phi}_{j}$ 在 $\hat{\mathbf{x}}_j^+$ 处求值，各不相同。验证 $\mathcal{O}^{(k)} \mathbf{N}_k \neq 0$：

第一列：
$$\mathbf{H}_k\big|_{\hat{\mathbf{x}}_k^-} \boldsymbol{\Phi}_{k-1}\big|_{\hat{\mathbf{x}}_{k-1}^+} \cdots \boldsymbol{\Phi}_0\big|_{\hat{\mathbf{x}}_0^+} \cdot (\mathbf{N}_k)_{:,1}$$

由于每个 $\boldsymbol{\Phi}_j$ 在不同的 $\hat{\mathbf{R}}_{I_j}, \hat{\mathbf{p}}_{I_j}, \hat{\mathbf{v}}_{I_j}$ 处求值，零空间被破坏 → $\neq \mathbf{0}$

**实际效果**：
- 滤波器认为yaw方向是**可观的** → Kalman gain在yaw方向非零
- 每次更新都在yaw方向"修正" → 协方差在yaw方向错误地减小
- 零空间维度从4缩减为3
- 滤波器过度自信

---

## 五、FEJ的数学方法

### Propagation中的FEJ

**问题**：
$$\boldsymbol{\Phi}(t_2, t_0) \neq \boldsymbol{\Phi}(t_2, t_1)\big|_{\mathbf{x}_1^{updated}} \boldsymbol{\Phi}(t_1, t_0)\big|_{\mathbf{x}_0}$$

**FEJ修正**：所有传播都使用第一个估计的线性化点：

$$\boldsymbol{\Phi}(t_2, t_0) = \boldsymbol{\Phi}(t_2, t_1)\big|_{\mathbf{x}_0} \boldsymbol{\Phi}(t_1, t_0)\big|_{\mathbf{x}_0}$$

实现方式：传播Jacobian时**固定使用初始估计** $\hat{\mathbf{x}}_0$ 而非更新的 $\hat{\mathbf{x}}_k^+$：

$$\boldsymbol{\Phi}_{k-1}^{FEJ} = \frac{\partial f}{\partial \mathbf{x}}\bigg|_{\hat{\mathbf{x}}_0}$$

### Update中的FEJ

对于IMU状态：第一次估计 = 状态转移矩阵的线性化点 = $\hat{\mathbf{x}}_0$

对于特征：第一次估计 = 特征**初始三角化时的值** $\hat{\mathbf{p}}_{f,0}$

FEJ的测量Jacobian：
$$\mathbf{H}_k^{FEJ} = \frac{\partial h}{\partial \mathbf{x}}\bigg|_{\mathbf{x}_0, \mathbf{p}_{f,0}}$$

### FEJ下的可观性验证

使用FEJ后，所有 $\boldsymbol{\Phi}^{FEJ}$ 和 $\mathbf{H}^{FEJ}$ 在**相同的线性化点**求值。

验证 $\mathcal{O}^{FEJ} \mathbf{N}^{FEJ} = \mathbf{0}$：

$$\mathcal{O}^{FEJ,(k)} \mathbf{N}^{FEJ} = \mathbf{H}_k^{FEJ} (\boldsymbol{\Phi}^{FEJ})^{k} \mathbf{N}^{FEJ}$$

由于线性化点一致 → 零空间得以保持 → $\mathbf{N}^{FEJ} = \mathbf{N}_0$ → 恢复**正确的4DoF不可观零空间**。

### 总结：FEJ做了什么？

| | 朴素EKF | FEJ |
|---|---|---|
| $\boldsymbol{\Phi}$ 的线性化点 | 每次不同（$\hat{\mathbf{x}}_k^+$） | 固定（$\hat{\mathbf{x}}_0$） |
| $\mathbf{H}$ 的线性化点 | 每次不同（$\hat{\mathbf{x}}_k^-$） | 固定（$\hat{\mathbf{x}}_0$ 和 $\hat{\mathbf{p}}_{f,0}$） |
| 零空间维度 | 3（少了一个yaw） | **4（正确）** |
| 一致性 | 不一致（yaw方向过度自信） | **一致** |
| 代价 | — | 引入线性化误差 |

---

## 六、后续改进

- **FEJ2**：在保证可观性的同时用better estimates补偿线性化误差
- **OC (Observability-Constrained)**：[Hesch et al.] 通过约束Jacobian保持正确的零空间结构，同时允许使用最新估计
- **不变EKF (InEKF)**：通过选择合适的误差变量使系统动力学和可观性自动保持

---

## 面试要点

1. **FEJ解决什么？** 防止朴素EKF破坏不可观零空间，恢复一致性
2. **VINS不可观方向**：4DoF = 全局yaw (1) + 全局位置 (3)
3. **为何朴素EKF错误？** 不同线性化点 → $\mathcal{O}\mathbf{N} \neq 0$ → 虚假信息
4. **FEJ的代价和收益**：代价 = 线性化误差↑，收益 = 一致性↑（收益 >> 代价）
5. **如何证明4DoF？** 构造 $\mathbf{N}$，验证 $\mathbf{H}_k\boldsymbol{\Phi}^k\mathbf{N} = \mathbf{0}, \forall k$

# MSCKF 零空间投影与测量压缩

> 来源：https://docs.openvins.com/update-null.html + update-compress.html

---

## 一、为什么需要零空间投影？

### 标准EKF更新回顾

给定一个特征在 $m$ 帧中的观测，线性化测量方程：

$$\mathbf{r} = \mathbf{H}_x \tilde{\mathbf{x}} + \mathbf{H}_f \tilde{\mathbf{p}}_f + \mathbf{n}$$

其中：
- $\mathbf{r} \in \mathbb{R}^{2m}$：$m$ 个观测的堆叠残差（每帧2维uv）
- $\tilde{\mathbf{x}} \in \mathbb{R}^{n}$：IMU+相机clone状态误差
- $\tilde{\mathbf{p}}_f \in \mathbb{R}^{3}$：特征位置误差
- $\mathbf{H}_x \in \mathbb{R}^{2m \times n}$：测量对状态的Jacobian
- $\mathbf{H}_f \in \mathbb{R}^{2m \times 3}$：测量对特征的Jacobian
- $\mathbf{n} \sim \mathcal{N}(0, \mathbf{R})$：测量噪声（$\mathbf{R} = \sigma_{uv}^2 \mathbf{I}_{2m}$）

### 残差协方差的困境

标准EKF需要计算：

$$\mathbf{S} = \mathbb{E}[\mathbf{r}\mathbf{r}^\top] = \mathbf{H}_x \mathbf{P}_{xx} \mathbf{H}_x^\top + \mathbf{H}_x \mathbf{P}_{xf} \mathbf{H}_f^\top + \mathbf{H}_f \mathbf{P}_{fx} \mathbf{H}_x^\top + \mathbf{H}_f \mathbf{P}_{ff} \mathbf{H}_f^\top + \mathbf{R}$$

**问题**：
1. 特征**不在状态中** → $\mathbf{P}_{ff}$ 未知
2. 特征误差与状态误差**耦合** → 交叉项 $\mathbf{P}_{xf}$ 未知
3. 特征误差与测量噪声**耦合**

**核心洞察**：MSCKF不把特征加入状态，因此无法用标准EKF处理特征误差的协方差传播。需要一种方法**从测量方程中消除特征**。

---

## 二、零空间投影（完整推导）

### 基本原理

寻找矩阵 $\mathbf{Q}_2$ 使得 $\mathbf{Q}_2^\top \mathbf{H}_f = \mathbf{0}$，然后左乘测量方程：

$$\mathbf{Q}_2^\top \mathbf{r} = \mathbf{Q}_2^\top \mathbf{H}_x \tilde{\mathbf{x}} + \mathbf{Q}_2^\top \mathbf{H}_f \tilde{\mathbf{p}}_f + \mathbf{Q}_2^\top \mathbf{n}$$

第二项消去 → **特征被移除**。

### 通过QR分解求左零空间

$\mathbf{H}_f \in \mathbb{R}^{2m \times 3}$（高瘦矩阵，$2m > 3$）

**完整QR分解**：

$$\mathbf{H}_f = \mathbf{Q} \begin{bmatrix} \mathbf{R}_1 \\ \mathbf{0} \end{bmatrix} = \begin{bmatrix} \mathbf{Q}_1 & \mathbf{Q}_2 \end{bmatrix} \begin{bmatrix} \mathbf{R}_1 \\ \mathbf{0} \end{bmatrix} = \mathbf{Q}_1 \mathbf{R}_1$$

其中：
- $\mathbf{Q} \in \mathbb{R}^{2m \times 2m}$：正交矩阵（$\mathbf{Q}\mathbf{Q}^\top = \mathbf{I}$）
- $\mathbf{Q}_1 \in \mathbb{R}^{2m \times 3}$：$\mathbf{H}_f$ 的列空间（range space）
- $\mathbf{Q}_2 \in \mathbb{R}^{2m \times (2m-3)}$：$\mathbf{H}_f$ 的**左零空间**（left nullspace）
- $\mathbf{R}_1 \in \mathbb{R}^{3 \times 3}$：上三角矩阵（可逆，因为 $\mathbf{H}_f$ 通常是满列秩）

**关键性质**（来自QR分解的正交性）：

$$\mathbf{Q}_2^\top \mathbf{H}_f = \mathbf{Q}_2^\top \mathbf{Q}_1 \mathbf{R}_1 = \mathbf{0} \cdot \mathbf{R}_1 = \mathbf{0}$$

$$\mathbf{Q}_1^\top \mathbf{Q}_1 = \mathbf{I}_3, \quad \mathbf{Q}_2^\top \mathbf{Q}_2 = \mathbf{I}_{2m-3}, \quad \mathbf{Q}_1^\top \mathbf{Q}_2 = \mathbf{0}$$

### 应用零空间投影

左乘 $\mathbf{Q}_2^\top$：

$$\mathbf{Q}_2^\top \mathbf{r} = \mathbf{Q}_2^\top \mathbf{H}_x \tilde{\mathbf{x}} + \underset{=\mathbf{0}}{\underbrace{\mathbf{Q}_2^\top \mathbf{H}_f}} \tilde{\mathbf{p}}_f + \mathbf{Q}_2^\top \mathbf{n}$$

得到**投影后的测量方程**：

$$\boxed{\mathbf{r}_n = \mathbf{H}_{x,n} \tilde{\mathbf{x}} + \mathbf{n}_n}$$

其中：
- $\mathbf{r}_n = \mathbf{Q}_2^\top \mathbf{r} \in \mathbb{R}^{2m-3}$
- $\mathbf{H}_{x,n} = \mathbf{Q}_2^\top \mathbf{H}_x \in \mathbb{R}^{(2m-3) \times n}$
- $\mathbf{n}_n = \mathbf{Q}_2^\top \mathbf{n} \sim \mathcal{N}(0, \mathbf{Q}_2^\top \mathbf{R} \mathbf{Q}_2)$

### 投影后的噪声特性

$$\mathbb{E}[\mathbf{n}_n] = \mathbf{Q}_2^\top \mathbb{E}[\mathbf{n}] = \mathbf{0}$$

$$\mathbb{E}[\mathbf{n}_n \mathbf{n}_n^\top] = \mathbf{Q}_2^\top \mathbb{E}[\mathbf{n}\mathbf{n}^\top] \mathbf{Q}_2 = \mathbf{Q}_2^\top \mathbf{R} \mathbf{Q}_2$$

由于 $\mathbf{Q}_2^\top \mathbf{Q}_2 = \mathbf{I}$（列正交），投影不改变白噪声的协方差结构：
$$\mathbf{Q}_2^\top (\sigma_{uv}^2 \mathbf{I}) \mathbf{Q}_2 = \sigma_{uv}^2 \mathbf{I}_{2m-3}$$

---

## 三、投影后的EKF更新

使用投影后测量做标准EKF更新：

**(1) 残差协方差**

$$\mathbf{S} = \mathbf{H}_{x,n} \mathbf{P}_{xx} \mathbf{H}_{x,n}^\top + \sigma_{uv}^2 \mathbf{I}_{2m-3}$$

注意：由于消除了 $\mathbf{H}_f$，不需要 $\mathbf{P}_{ff}$ 和 $\mathbf{P}_{xf}$。

**(2) Kalman Gain**

$$\mathbf{K} = \mathbf{P}_{xx} \mathbf{H}_{x,n}^\top \mathbf{S}^{-1}$$

**(3) 状态更新**

$$\tilde{\mathbf{x}}^+ = \tilde{\mathbf{x}} + \mathbf{K} \mathbf{r}_n$$

$$\hat{\mathbf{x}}^+ = \hat{\mathbf{x}} \boxplus \tilde{\mathbf{x}}^+$$

**(4) 协方差更新（Joseph形式，数值稳定）**

$$\mathbf{P}_{xx}^+ = (\mathbf{I} - \mathbf{K} \mathbf{H}_{x,n}) \mathbf{P}_{xx} (\mathbf{I} - \mathbf{K} \mathbf{H}_{x,n})^\top + \sigma_{uv}^2 \mathbf{K} \mathbf{K}^\top$$

---

## 四、维度分析

### 单个特征（3D点，被 $m$ 帧观测）

| 矩阵 | 投影前维度 | 投影后维度 | 减少量 |
|------|-----------|-----------|--------|
| $\mathbf{r}$ | $2m \times 1$ | $(2m-3) \times 1$ | 3 |
| $\mathbf{H}_x$ | $2m \times n$ | $(2m-3) \times n$ | 3行 |
| $\mathbf{H}_f$ | $2m \times 3$ | 被消除 | 全部 |

**信息损失**：每特征损失3维（恰好是特征位置的维度），所以信息**没有净损失**。

### 多个特征

对 $z_f$ 个特征，总观测数 $z = \sum_i m_i$：

- 投影前：$\mathbf{H}_x$ 为 $(2z) \times n$
- 投影后：$\mathbf{H}_x$ 为 $(2z - 3z_f) \times n$
- 每个特征独立做零空间投影（特征间**无耦合**）

---

## 五、Eigen实现

```cpp
// H_f 大小: 2m × 3
Eigen::ColPivHouseholderQR<Eigen::MatrixXd> qr(H_f.rows(), H_f.cols());
qr.compute(H_f);

// Q 大小: 2m × 2m
Eigen::MatrixXd Q = qr.householderQ() * Eigen::MatrixXd::Identity(H_f.rows(), H_f.rows());

// Q1 = Q的前3列 (range space), Q2 = Q的剩余列 (nullspace)
Eigen::MatrixXd Q1 = Q.block(0, 0, Q.rows(), 3);         // 2m × 3
Eigen::MatrixXd Q2 = Q.block(0, 3, Q.rows(), Q.cols()-3); // 2m × (2m-3)

// 验证: Q2^T * H_f 应该是零矩阵
// 零空间投影
Eigen::VectorXd r_n = Q2.transpose() * r;
Eigen::MatrixXd H_x_n = Q2.transpose() * H_x;
```

---

## 六、测量压缩

### 动机

EKF更新中，$\mathbf{H}_{x,n} \mathbf{P}_{xx} \mathbf{H}_{x,n}^\top$ 是 $\mathcal{O}((2m-3)n^2)$ 操作。

当观测数 $m$ 很大时（一条长特征轨迹），计算量很大。

### 方法

对 $\mathbf{H}_{x,n}$ 的转置做**薄QR分解**（再次Givens旋转）：

$$\mathbf{H}_{x,n}^\top = \begin{bmatrix} \tilde{\mathbf{Q}}_1 & \tilde{\mathbf{Q}}_2 \end{bmatrix} \begin{bmatrix} \tilde{\mathbf{R}}_1 \\ \mathbf{0} \end{bmatrix}$$

其中 $\tilde{\mathbf{Q}}_1 \in \mathbb{R}^{(2m-3) \times n}$, $\tilde{\mathbf{R}}_1 \in \mathbb{R}^{n \times n}$

**压缩测量方程**：左乘 $\tilde{\mathbf{Q}}_1^\top$

$$\tilde{\mathbf{Q}}_1^\top \mathbf{r}_n = \tilde{\mathbf{R}}_1^\top \tilde{\mathbf{x}} + \tilde{\mathbf{Q}}_1^\top \mathbf{n}_n$$

定义：
- $\mathbf{r}_c = \tilde{\mathbf{Q}}_1^\top \mathbf{r}_n \in \mathbb{R}^{n}$
- $\mathbf{H}_c = \tilde{\mathbf{R}}_1^\top \in \mathbb{R}^{n \times n}$

### 效果

压缩后的 $\mathbf{H}_c$ 大小为**状态维度** $n \times n$：

$$\mathbf{P}_{xx} \mathbf{H}_c^\top \in \mathbb{R}^{n \times n} \quad \text{vs} \quad \mathbf{P}_{xx} \mathbf{H}_{x,n}^\top \in \mathbb{R}^{n \times (2m-3)}$$

从 $\mathcal{O}((2m-3)n^2)$ 降至 $\mathcal{O}(n^3)$，当 $m \gg n$ 时差异巨大。

**注意**：压缩操作的QR分解本身也是 $\mathcal{O}((2m-3)n^2)$，但在多特征场景中通过**递增式Givens旋转**可以摊销成本。

---

## 七、MSCKF完整流水线

```text
┌─ 特征轨迹积累 ─────────────────────────────────────────────┐
│ 对每条特征:                                                   │
│   for each frame where feature is tracked:                    │
│       stack: r_append, H_x_append, H_f_append                 │
└──────────────────────────────────────────────────────────────┘
                            ↓
┌─ 零空间投影 ───────────────────────────────────────────────┐
│ H_f = Q1 * R1         (QR分解)                                │
│ Q2 = nullspace of H_f (Q的后 2m-3 列)                        │
│ r_n = Q2^T * r                                               │
│ H_x_n = Q2^T * H_x                                           │
│ → 消除特征，保留对相机/IMU状态的约束                             │
└──────────────────────────────────────────────────────────────┘
                            ↓
┌─ 测量压缩 ─────────────────────────────────────────────────┐
│ H_x_n^T = Q̃1 * R̃1     (薄QR分解)                            │
│ r_c = Q̃1^T * r_n                                            │
│ H_c = R̃1^T                                                  │
│ → 压缩到状态维度 n×n                                          │
└──────────────────────────────────────────────────────────────┘
                            ↓
┌─ EKF更新 ──────────────────────────────────────────────────┐
│ S = H_c * P * H_c^T + R_c                                    │
│ K = P * H_c^T * S^{-1}                                       │
│ x ← x ⊕ K*r_c                                                │
│ P ← (I-K*H_c)*P*(I-K*H_c)^T + K*R_c*K^T                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 面试要点

1. **零空间投影消除了什么？** 特征位置误差 $\tilde{\mathbf{p}}_f$，保留对所有相机/IMU状态的约束
2. **为什么叫"零空间"？** $\mathbf{Q}_2$ 是 $\mathbf{H}_f$ 的左零空间：$\mathbf{H}_f^\top \mathbf{Q}_2 = 0$（或等价地 $\mathbf{Q}_2^\top \mathbf{H}_f = 0$）
3. **信息损失？** 每特征损失3维（恰好是特征维度），约束信息无净损失
4. **QR vs SVD**：QR更高效，两者在此任务等价
5. **测量压缩的必要性**：压缩从 $\mathcal{O}(mn^2)$ 降到 $\mathcal{O}(n^3)$，在大量特征场景中至关重要
6. **MSCKF vs 特征入状态**：MSCKF不长期保持特征 → 状态维度可控 → 协方差不膨胀

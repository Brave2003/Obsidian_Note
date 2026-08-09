# IMU 传播推导 (IMU Propagation Derivations)

> 来源：https://docs.openvins.com/propagation.html

---

## 一、IMU 测量模型

使用6轴IMU传播惯性导航系统（INS）：

**陀螺仪测量**：
$$\boldsymbol{\omega}_m(t) = \boldsymbol{\omega}(t) + \mathbf{b}_g(t) + \mathbf{n}_g(t)$$

**加速度计测量**：
$$\mathbf{a}_m(t) = \mathbf{a}(t) + \mathbf{R}_{IG}(t) \mathbf{g} + \mathbf{b}_a(t) + \mathbf{n}_a(t)$$

其中：
- $\boldsymbol{\omega}, \mathbf{a}$：IMU局部坐标系 $\{I\}$ 下的真实角速度和加速度
- $\mathbf{b}_g, \mathbf{b}_a$：随时间缓变的bias
- $\mathbf{n}_g, \mathbf{n}_a$：白高斯噪声，$\mathbf{n}_g \sim \mathcal{N}(0, \sigma_g^2\mathbf{I})$, $\mathbf{n}_a \sim \mathcal{N}(0, \sigma_a^2\mathbf{I})$
- $\mathbf{g} = [0, 0, -9.81]^\top$：全局重力向量（不同地理位置略有差异）
- $\mathbf{R}_{IG}$：全局 $\{G\}$ 到IMU局部 $\{I\}$ 的旋转矩阵

> 注意：测量模型中的噪声在连续时间下是**功率谱密度**（PSD），离散实现时需转换为方差：$\sigma_d^2 = \sigma_c^2 / \Delta t$

---

## 二、惯性状态向量

### 名义状态

时刻 $t_k$ 的INS状态向量（JPL约定）：

$$\mathbf{x}_I(t_k) = \begin{bmatrix} {}_{G}^{I}\bar{q}(t_k) \\ {}^{G}\mathbf{p}_I(t_k) \\ {}^{G}\mathbf{v}_I(t_k) \\ \mathbf{b}_g(t_k) \\ \mathbf{b}_a(t_k) \end{bmatrix}$$

其中：
- ${}_{G}^{I}\bar{q}$：全局→IMU的单位四元数（JPL约定：$\bar{q} = [q_x, q_y, q_z, q_w]^\top$）
- ${}^{G}\mathbf{p}_I$：IMU在全局坐标系 $\{G\}$ 中的位置
- ${}^{G}\mathbf{v}_I$：IMU在全局坐标系 $\{G\}$ 中的速度
- $\mathbf{b}_g, \mathbf{b}_a$：陀螺仪和加速度计bias

### 误差状态定义

位置、速度、bias使用**标准加法误差**：

$$\mathbf{p} = \hat{\mathbf{p}} + \tilde{\mathbf{p}}, \quad \mathbf{v} = \hat{\mathbf{v}} + \tilde{\mathbf{v}}, \quad \mathbf{b} = \hat{\mathbf{b}} + \tilde{\mathbf{b}}$$

姿态使用**左乘四元数误差**（JPL约定下）：

$${}_{G}^{I}\bar{q} = \delta\bar{q} \otimes {}_{G}^{I}\hat{\bar{q}}$$

其中 $\delta\bar{q}$ 是小旋转：

$$\delta\bar{q} = \begin{bmatrix} \hat{\mathbf{k}} \sin(\delta\theta/2) \\ \cos(\delta\theta/2) \end{bmatrix} \approx \begin{bmatrix} \frac{1}{2}\delta\boldsymbol{\theta} \\ 1 \end{bmatrix}$$

这里 $\delta\boldsymbol{\theta} \in \mathbb{R}^3$ 是局部的姿态误差角。

**完整的15维误差状态**（不是16维）：

$$\tilde{\mathbf{x}}_I = \begin{bmatrix} {}^{I}\delta\boldsymbol{\theta} \\ {}^{G}\tilde{\mathbf{p}}_I \\ {}^{G}\tilde{\mathbf{v}}_I \\ \tilde{\mathbf{b}}_g \\ \tilde{\mathbf{b}}_a \end{bmatrix} \in \mathbb{R}^{15}$$

> **为什么15维？** 四元数4参数但旋转仅3自由度。误差用3维小角度表示，避免了四元数的单位模约束。

---

## 三、IMU运动学方程（连续时间）

### 真实状态的运动学

四元数动力学（JPL约定下）：

$$\dot{\bar{q}}_{IG} = \frac{1}{2} \begin{bmatrix} -\lfloor \boldsymbol{\omega} \times \rfloor & \boldsymbol{\omega} \\ -\boldsymbol{\omega}^\top & 0 \end{bmatrix} \bar{q}_{IG} = \frac{1}{2} \boldsymbol{\Omega}(\boldsymbol{\omega}) \bar{q}_{IG}$$

其中 $\lfloor \cdot \times \rfloor$ 是反对称矩阵算子：

$$\lfloor \boldsymbol{\omega} \times \rfloor = \begin{bmatrix} 0 & -\omega_z & \omega_y \\ \omega_z & 0 & -\omega_x \\ -\omega_y & \omega_x & 0 \end{bmatrix}$$

位置和速度：

$$\dot{\mathbf{p}}_I = \mathbf{v}_I$$

$$\dot{\mathbf{v}}_I = \mathbf{R}_{IG}^\top \mathbf{a} + \mathbf{g} = {}^{I}\mathbf{a} + \mathbf{g}$$

bias（建模为随机游走）：

$$\dot{\mathbf{b}}_g(t) = \mathbf{n}_{wg}(t), \quad \dot{\mathbf{b}}_a(t) = \mathbf{n}_{wa}(t)$$

其中 $\mathbf{n}_{wg} \sim \mathcal{N}(0, \sigma_{wg}^2\mathbf{I})$, $\mathbf{n}_{wa} \sim \mathcal{N}(0, \sigma_{wa}^2\mathbf{I})$

### 名义状态的运动学（用于传播）

将测量代入：

$$\dot{\hat{\bar{q}}}_{IG} = \frac{1}{2} \boldsymbol{\Omega}(\boldsymbol{\omega}_m - \hat{\mathbf{b}}_g) \hat{\bar{q}}_{IG}$$

$$\dot{\hat{\mathbf{p}}}_I = \hat{\mathbf{v}}_I$$

$$\dot{\hat{\mathbf{v}}}_I = \hat{\mathbf{R}}_{IG}^\top (\mathbf{a}_m - \hat{\mathbf{b}}_a) + \mathbf{g}$$

$$\dot{\hat{\mathbf{b}}}_g = \mathbf{0}, \quad \dot{\hat{\mathbf{b}}}_a = \mathbf{0}$$

### 误差状态动力学推导

**(1) 姿态误差动力学**

从 $\bar{q} = \delta\bar{q} \otimes \hat{\bar{q}}$ 求导：

$$\dot{\bar{q}} = \dot{\delta\bar{q}} \otimes \hat{\bar{q}} + \delta\bar{q} \otimes \dot{\hat{\bar{q}}}$$

代入真实的四元数动力学和名义的动力学，利用 $\boldsymbol{\omega} = \boldsymbol{\omega}_m - \mathbf{b}_g - \mathbf{n}_g$ 和 $\hat{\boldsymbol{\omega}} = \boldsymbol{\omega}_m - \hat{\mathbf{b}}_g$，经过推导可得：

$$\dot{\delta\boldsymbol{\theta}} = -\lfloor \hat{\boldsymbol{\omega}} \times \rfloor \delta\boldsymbol{\theta} - \tilde{\mathbf{b}}_g - \mathbf{n}_g$$

**(2) 位置误差动力学**

$$\dot{\tilde{\mathbf{p}}}_I = \dot{\mathbf{p}}_I - \dot{\hat{\mathbf{p}}}_I = \tilde{\mathbf{v}}_I$$

**(3) 速度误差动力学**

$$\dot{\tilde{\mathbf{v}}}_I = \dot{\mathbf{v}}_I - \dot{\hat{\mathbf{v}}}_I$$
$$= (\mathbf{R}_{IG}^\top \mathbf{a} + \mathbf{g}) - (\hat{\mathbf{R}}_{IG}^\top (\mathbf{a}_m - \hat{\mathbf{b}}_a) + \mathbf{g})$$

利用 $\mathbf{R} = (\mathbf{I} - \lfloor \delta\boldsymbol{\theta} \times \rfloor)\hat{\mathbf{R}}$（小角度近似）：

$$\dot{\tilde{\mathbf{v}}}_I = -\hat{\mathbf{R}}_{IG}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a) \times \rfloor \delta\boldsymbol{\theta} - \hat{\mathbf{R}}_{IG}^\top \tilde{\mathbf{b}}_a - \hat{\mathbf{R}}_{IG}^\top \mathbf{n}_a$$

**(4) Bias误差动力学**

$$\dot{\tilde{\mathbf{b}}}_g = \mathbf{n}_{wg}, \quad \dot{\tilde{\mathbf{b}}}_a = \mathbf{n}_{wa}$$

### 误差状态系统矩阵

写成紧凑形式：$\dot{\tilde{\mathbf{x}}} = \mathbf{F} \tilde{\mathbf{x}} + \mathbf{G} \mathbf{n}$

$$\mathbf{F} = \begin{bmatrix}
-\lfloor \hat{\boldsymbol{\omega}} \times \rfloor & \mathbf{0} & \mathbf{0} & -\mathbf{I}_{3\times3} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{I}_{3\times3} & \mathbf{0} & \mathbf{0} \\
-\hat{\mathbf{R}}_{IG}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a) \times \rfloor & \mathbf{0} & \mathbf{0} & \mathbf{0} & -\hat{\mathbf{R}}_{IG}^\top \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0}
\end{bmatrix}$$

$$\mathbf{G} = \begin{bmatrix}
-\mathbf{I} & \mathbf{0} & \mathbf{0} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0} \\
\mathbf{0} & -\hat{\mathbf{R}}_{IG}^\top & \mathbf{0} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{I} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{I}
\end{bmatrix}, \quad \mathbf{n} = \begin{bmatrix} \mathbf{n}_g \\ \mathbf{n}_a \\ \mathbf{n}_{wg} \\ \mathbf{n}_{wa} \end{bmatrix}$$

---

## 四、离散传播

### 名义状态传播（RK4积分）

从 $t_k$ 到 $t_{k+1}$，$\Delta t = t_{k+1} - t_k$：

**姿态**：
$$\hat{\mathbf{R}}_{I_{k+1}} = \hat{\mathbf{R}}_{I_k} \exp\left((\boldsymbol{\omega}_m - \hat{\mathbf{b}}_g) \Delta t\right)^\wedge$$

**位置**：
$$\hat{\mathbf{p}}_{I_{k+1}} = \hat{\mathbf{p}}_{I_k} + \hat{\mathbf{v}}_{I_k} \Delta t + \frac{1}{2}(\hat{\mathbf{R}}_{I_k}^\top (\mathbf{a}_m - \hat{\mathbf{b}}_a) + \mathbf{g}) \Delta t^2$$

**速度**：
$$\hat{\mathbf{v}}_{I_{k+1}} = \hat{\mathbf{v}}_{I_k} + (\hat{\mathbf{R}}_{I_k}^\top (\mathbf{a}_m - \hat{\mathbf{b}}_a) + \mathbf{g}) \Delta t$$

### 误差状态协方差传播

**(1) 状态转移矩阵** $\boldsymbol{\Phi}(t_{k+1}, t_k)$

通过求解 $\dot{\boldsymbol{\Phi}} = \mathbf{F} \boldsymbol{\Phi}$（在恒定 $\mathbf{F}$ 假设下）：

$$\boldsymbol{\Phi} = \exp(\mathbf{F} \Delta t) = \mathbf{I} + \mathbf{F}\Delta t + \frac{1}{2!}\mathbf{F}^2\Delta t^2 + \cdots$$

展开为分块形式：

$$\boldsymbol{\Phi} = \begin{bmatrix}
\boldsymbol{\Phi}_{11} & \mathbf{0} & \mathbf{0} & \boldsymbol{\Phi}_{14} & \mathbf{0} \\
\boldsymbol{\Phi}_{21} & \mathbf{I} & \mathbf{I}\Delta t & \boldsymbol{\Phi}_{24} & \boldsymbol{\Phi}_{25} \\
\boldsymbol{\Phi}_{31} & \mathbf{0} & \mathbf{I} & \boldsymbol{\Phi}_{34} & \boldsymbol{\Phi}_{35} \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{I} & \mathbf{0} \\
\mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{0} & \mathbf{I}
\end{bmatrix}$$

各分块的推导：

**姿态误差自转移**：
$$\boldsymbol{\Phi}_{11} = \exp\left(-\lfloor \hat{\boldsymbol{\omega}} \times \rfloor \Delta t\right) \approx \mathbf{I} - \lfloor \hat{\boldsymbol{\omega}} \times \rfloor \Delta t$$

**姿态误差→bias**（陀螺仪bias对姿态的累积影响）：
$$\boldsymbol{\Phi}_{14} = -\mathbf{I} \Delta t - \frac{1}{2}\lfloor \hat{\boldsymbol{\omega}} \times \rfloor \Delta t^2 + \cdots$$

**位置→姿态**（姿态误差导致的位置漂移）：
$$\boldsymbol{\Phi}_{21} = -\lfloor \mathbf{g} \times \rfloor \hat{\mathbf{R}}_{I_k} \Delta t \cdot \frac{\partial \mathbf{p}}{\partial \delta\boldsymbol{\theta}}$$

完整推导：
$$\boldsymbol{\Phi}_{21} = -\frac{1}{2}\hat{\mathbf{R}}_{I_k}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a) \times \rfloor \Delta t^2$$

**位置→bias**：
$$\boldsymbol{\Phi}_{24} = -\frac{1}{2}\hat{\mathbf{R}}_{I_k}^\top \Delta t^2, \quad \boldsymbol{\Phi}_{25} = -\frac{1}{2}\hat{\mathbf{R}}_{I_k}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a) \times \rfloor \Delta t^2$$

**速度→姿态**：
$$\boldsymbol{\Phi}_{31} = -\hat{\mathbf{R}}_{I_k}^\top \lfloor (\mathbf{a}_m - \hat{\mathbf{b}}_a) \times \rfloor \Delta t$$

**速度→bias**：
$$\boldsymbol{\Phi}_{34} = -\hat{\mathbf{R}}_{I_k}^\top \Delta t, \quad \boldsymbol{\Phi}_{35} = 0$$

**(2) 离散噪声协方差矩阵**

$$\mathbf{Q}_d = \int_{t_k}^{t_{k+1}} \boldsymbol{\Phi}(t_{k+1}, \tau) \mathbf{G} \mathbf{Q}_c \mathbf{G}^\top \boldsymbol{\Phi}(t_{k+1}, \tau)^\top d\tau$$

其中连续噪声矩阵：
$$\mathbf{Q}_c = \text{diag}(\sigma_g^2\mathbf{I}, \sigma_a^2\mathbf{I}, \sigma_{wg}^2\mathbf{I}, \sigma_{wa}^2\mathbf{I})$$

近似计算（假设 $\boldsymbol{\Phi} \approx \mathbf{I}$ 在短时间段）：
$$\mathbf{Q}_d \approx \mathbf{G} \mathbf{Q}_c \mathbf{G}^\top \Delta t$$

**(3) 协方差传播**

$$\mathbf{P}_{k+1|k} = \boldsymbol{\Phi}(t_{k+1}, t_k) \mathbf{P}_{k|k} \boldsymbol{\Phi}(t_{k+1}, t_k)^\top + \mathbf{Q}_d$$

---

## 五、解析传播 vs 离散传播

| | 离散传播 | 解析传播 |
|---|---|---|
| 计算方式 | $\boldsymbol{\Phi} = \mathbf{I} + \mathbf{F}\Delta t$ | $\boldsymbol{\Phi} = \exp(\mathbf{F}\Delta t)$ 闭式解 |
| 精度 | $\mathcal{O}(\Delta t^2)$ | 精确（在恒定测量假设下） |
| IMU内参 | 不考虑 | 可以包含 $\mathbf{T}_g, \mathbf{T}_a$ 等 |
| 计算量 | 低 | 中 |
| 适用场景 | 高频率IMU（$\Delta t$ 很小） | 较低频率或高动态 |

---

## 六、IMU Propagation vs IMU Preintegration

**重要区分**：OpenVINS使用IMU进行**传播**而非预积分：

| | IMU Propagation (EKF) | IMU Preintegration (优化) |
|---|---|---|
| 目标 | 传播状态和协方差 | 构造IMU残差因子 |
| 框架 | Kalman滤波 | 因子图优化 |
| 状态 | 实时维护 $\mathbf{P}$ | 预积分协方差 $\boldsymbol{\Sigma}_{ij}$ |
| bias修正 | 重新传播 | 一阶近似 $\partial \Delta / \partial \mathbf{b}$ |
| 线性化 | 在线性化点处 | 在预积分时固定 |

---

## 面试要点

1. **为什么误差状态15维？** 四元数4参数 → 旋转3自由度 → 误差用3维角度
2. **名义状态 vs 误差状态传播**：名义用非线性RK4，误差用线性状态转移矩阵
3. **bias建模**：随机游走 $\dot{\mathbf{b}} = \mathbf{n}_w$，$\mathbf{n}_w$ 是白噪声
4. **连续→离散噪声转换**：白噪声方差 $\sigma_d^2 = \sigma_c^2 / \Delta t$，随机游走方差 $\sigma_d^2 = \sigma_c^2 \cdot \Delta t$
5. **$\boldsymbol{\Phi}_{21}$ 的物理含义**：姿态误差通过重力耦合到位置误差

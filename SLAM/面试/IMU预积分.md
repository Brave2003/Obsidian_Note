好的，我来为你详细推导IMU的运动学模型以及IMU预积分的核心公式。这部分是VIO系统的数学基础，面试中如果问到“IMU预积分是怎么来的”或者“为什么需要预积分”，你就可以用这些公式来回答。

---

## IMU运动学模型与预积分公式推导

---

### 第一部分：IMU测量模型

IMU（陀螺仪+加速度计）的原始测量值为角速度 $\boldsymbol{\omega}$ 和加速度 $\mathbf{a}$。真实值与测量值之间的关系为：

$$
\begin{aligned}
\boldsymbol{\omega}_m(t) &= \boldsymbol{\omega}(t) + \mathbf{b}_g(t) + \boldsymbol{\eta}_g(t) \\
\mathbf{a}_m(t) &= \mathbf{a}(t) + \mathbf{b}_a(t) + \boldsymbol{\eta}_a(t)
\end{aligned}
$$

其中：
- $\boldsymbol{\omega}_m$、$\mathbf{a}_m$：IMU的测量值
- $\boldsymbol{\omega}$、$\mathbf{a}$：真实的角速度和加速度
- $\mathbf{b}_g$、$\mathbf{b}_a$：陀螺仪和加速度计的**零偏（bias）**，随时间缓慢变化
- $\boldsymbol{\eta}_g$、$\boldsymbol{\eta}_a$：高斯白噪声

加速度的真实值 $\mathbf{a}(t)$ 与载体运动之间的关系为：

$$
\mathbf{a}(t) = {}_W^I \mathbf{R}(t) \left( {}^W \ddot{\mathbf{p}}(t) - {}^W \mathbf{g} \right)
$$

其中：
- ${}_W^I \mathbf{R}(t)$ 表示从世界坐标系 $W$ 到IMU坐标系 $I$ 的旋转矩阵
- ${}^W \ddot{\mathbf{p}}(t)$ 是IMU在世界坐标系下的加速度
- ${}^W \mathbf{g}$ 是世界坐标系下的重力向量

---

### 第二部分：连续时间运动学模型

IMU的位姿状态可以表示为：

$$
\mathbf{x}(t) = \left[ {}_W^I \mathbf{R}(t), \ {}^W \mathbf{p}(t), \ {}^W \mathbf{v}(t), \ \mathbf{b}_g(t), \ \mathbf{b}_a(t) \right]^T
$$

其中 ${}^W \mathbf{p}$ 和 ${}^W \mathbf{v}$ 分别是IMU在世界坐标系下的位置和速度。

**连续时间下的导数关系**：

$$
\begin{aligned}
{}_W^I \dot{\mathbf{R}}(t) &= {}_W^I \mathbf{R}(t) \left( \boldsymbol{\omega}_m(t) - \mathbf{b}_g(t) - \boldsymbol{\eta}_g(t) \right)^\wedge \\
{}^W \dot{\mathbf{p}}(t) &= {}^W \mathbf{v}(t) \\
{}^W \dot{\mathbf{v}}(t) &= {}_I^W \mathbf{R}(t) \left( \mathbf{a}_m(t) - \mathbf{b}_a(t) - \boldsymbol{\eta}_a(t) \right) + {}^W \mathbf{g} \\
\dot{\mathbf{b}}_g(t) &= \boldsymbol{\eta}_{bg}(t) \\
\dot{\mathbf{b}}_a(t) &= \boldsymbol{\eta}_{ba}(t)
\end{aligned}
$$

其中 $(\cdot)^\wedge$ 表示向量的反对称矩阵，$\boldsymbol{\eta}_{bg}$、$\boldsymbol{\eta}_{ba}$ 是偏置的随机游走噪声。

> **注意**：旋转矩阵的时间导数公式为 $\dot{\mathbf{R}} = \mathbf{R} \boldsymbol{\omega}^\wedge$，其中 $\boldsymbol{\omega}$ 是在**IMU坐标系**下表示的角速度。

---

### 第三部分：为什么要做IMU预积分？

在VIO系统中，后端优化需要计算两帧图像 $i$ 和 $j$ 之间的IMU残差。

**传统方法的问题**：
- 如果直接使用原始IMU测量值，状态更新时需要**重新积分所有IMU数据**
- 每次优化迭代时，状态量（如位姿、速度、偏置）发生变化，都需要重新积分，计算量随IMU频率（200-1000Hz）和关键帧数量增长

**预积分的核心思想**：
- 将两帧之间的所有IMU测量值**预先积分为一个整体**，得到一个**相对运动增量**
- 这个增量只依赖于IMU测量值和偏置，与绝对位姿无关
- 优化时只需计算**预积分残差**，无需重新积分所有测量值

---

### 第四部分：预积分公式推导

将连续时间运动学模型在时间段 $[t_i, t_j]$ 上积分：

#### 1. 位置预积分

从速度的积分开始：

$$
{}^W \mathbf{p}_j = {}^W \mathbf{p}_i + {}^W \mathbf{v}_i \Delta t_{ij} + \iint_{t_i}^{t_j} \left( {}_I^W \mathbf{R}_t \left( \mathbf{a}_m(t) - \mathbf{b}_a(t) - \boldsymbol{\eta}_a(t) \right) + {}^W \mathbf{g} \right) dt^2
$$

#### 2. 速度预积分

$$
{}^W \mathbf{v}_j = {}^W \mathbf{v}_i + \int_{t_i}^{t_j} \left( {}_I^W \mathbf{R}_t \left( \mathbf{a}_m(t) - \mathbf{b}_a(t) - \boldsymbol{\eta}_a(t) \right) + {}^W \mathbf{g} \right) dt
$$

#### 3. 旋转预积分

$$
{}_W^I \mathbf{R}_j = {}_W^I \mathbf{R}_i \cdot \Delta \mathbf{R}_{ij}
$$

其中：

$$
\Delta \mathbf{R}_{ij} = \int_{t_i}^{t_j} \Delta \mathbf{R}_{it} \left( \boldsymbol{\omega}_m(t) - \mathbf{b}_g(t) - \boldsymbol{\eta}_g(t) \right)^\wedge dt
$$

> **关键观察**：预积分量 $\Delta \mathbf{R}_{ij}$ 只依赖于IMU测量值和偏置，**与绝对位姿无关**。

---

### 第五部分：提取预积分项

为了将重力项和绝对位姿分离，我们定义三个**预积分量**：

$$
\begin{aligned}
\Delta \mathbf{R}_{ij} &\triangleq \prod_{k=i}^{j-1} \exp \left( \left( \boldsymbol{\omega}_{m,k} - \mathbf{b}_{g,i} - \boldsymbol{\eta}_{g,k} \right)^\wedge \Delta t \right) \\
\Delta \mathbf{v}_{ij} &\triangleq \sum_{k=i}^{j-1} \Delta \mathbf{R}_{ik} \left( \mathbf{a}_{m,k} - \mathbf{b}_{a,i} - \boldsymbol{\eta}_{a,k} \right) \Delta t \\
\Delta \mathbf{p}_{ij} &\triangleq \sum_{k=i}^{j-1} \left( \Delta \mathbf{v}_{ik} \Delta t + \frac{1}{2} \Delta \mathbf{R}_{ik} \left( \mathbf{a}_{m,k} - \mathbf{b}_{a,i} - \boldsymbol{\eta}_{a,k} \right) \Delta t^2 \right)
\end{aligned}
$$

这里用离散形式表示，$\Delta t$ 是相邻两个IMU测量之间的时间间隔。

- $\Delta \mathbf{R}_{ij}$：旋转预积分量，表示从时刻 $i$ 到 $j$ 的旋转变化
- $\Delta \mathbf{v}_{ij}$：速度预积分量，表示从时刻 $i$ 到 $j$ 的速度变化（排除重力影响）
- $\Delta \mathbf{p}_{ij}$：位置预积分量，表示从时刻 $i$ 到 $j$ 的位置变化（排除重力影响）

**注意**：预积分量是在**IMU坐标系 $i$ 时刻**下计算的，而不是在世界坐标系下。

---

### 第六部分：预积分测量模型（带有噪声和偏置）

实际工程中，预积分量会受到噪声 $\boldsymbol{\eta}$ 和偏置 $\mathbf{b}$ 的影响。将噪声项分离后，预积分量的**真实值**可以表示为：

$$
\begin{aligned}
\Delta \tilde{\mathbf{R}}_{ij} &\approx \Delta \mathbf{R}_{ij} \exp \left( - \frac{\partial \Delta \mathbf{R}_{ij}}{\partial \mathbf{b}_g} \delta \mathbf{b}_{g,i} - \boldsymbol{\eta}_{ij}^{\Delta R} \right) \\
\Delta \tilde{\mathbf{v}}_{ij} &\approx \Delta \mathbf{v}_{ij} - \frac{\partial \Delta \mathbf{v}_{ij}}{\partial \mathbf{b}_g} \delta \mathbf{b}_{g,i} - \frac{\partial \Delta \mathbf{v}_{ij}}{\partial \mathbf{b}_a} \delta \mathbf{b}_{a,i} - \boldsymbol{\eta}_{ij}^{\Delta v} \\
\Delta \tilde{\mathbf{p}}_{ij} &\approx \Delta \mathbf{p}_{ij} - \frac{\partial \Delta \mathbf{p}_{ij}}{\partial \mathbf{b}_g} \delta \mathbf{b}_{g,i} - \frac{\partial \Delta \mathbf{p}_{ij}}{\partial \mathbf{b}_a} \delta \mathbf{b}_{a,i} - \boldsymbol{\eta}_{ij}^{\Delta p}
\end{aligned}
$$

其中 $\delta \mathbf{b}_{g,i}$ 和 $\delta \mathbf{b}_{a,i}$ 是偏置的变化量，$\boldsymbol{\eta}_{ij}^{\Delta R}$、$\boldsymbol{\eta}_{ij}^{\Delta v}$、$\boldsymbol{\eta}_{ij}^{\Delta p}$ 是累积噪声项（预积分噪声）。

**重要结论**：
- 预积分量可以用**当前估计的偏置**重新传播，也可以用**一阶泰勒展开**快速更新
- 当偏置变化较小时（如优化迭代中），用一阶近似更新预积分量，无需重新积分

---

### 第七部分：预积分残差（用于后端优化）

在滑动窗口优化中，IMU残差定义为**预积分测量值与估计值之间的差异**：

$$
\mathbf{r}_{\text{IMU}} = \begin{bmatrix}
\mathbf{r}_p \\
\mathbf{r}_v \\
\mathbf{r}_R \\
\mathbf{r}_{bg} \\
\mathbf{r}_{ba}
\end{bmatrix}
$$

其中：

$$
\begin{aligned}
\mathbf{r}_p &= {}_I^W \mathbf{R}_i^T \left( {}^W \mathbf{p}_j - {}^W \mathbf{p}_i - {}^W \mathbf{v}_i \Delta t_{ij} - \frac{1}{2} {}^W \mathbf{g} \Delta t_{ij}^2 \right) - \Delta \tilde{\mathbf{p}}_{ij} \\
\mathbf{r}_v &= {}_I^W \mathbf{R}_i^T \left( {}^W \mathbf{v}_j - {}^W \mathbf{v}_i - {}^W \mathbf{g} \Delta t_{ij} \right) - \Delta \tilde{\mathbf{v}}_{ij} \\
\mathbf{r}_R &= \log \left( \Delta \tilde{\mathbf{R}}_{ij}^T \cdot {}_I^W \mathbf{R}_i^T \cdot {}_I^W \mathbf{R}_j \right)^\vee \\
\mathbf{r}_{bg} &= \mathbf{b}_{g,j} - \mathbf{b}_{g,i} \\
\mathbf{r}_{ba} &= \mathbf{b}_{a,j} - \mathbf{b}_{a,i}
\end{aligned}
$$

> **符号说明**：
> - $\log(\cdot)^\vee$：将旋转矩阵映射到旋转向量（李群到李代数的对数映射）
> - $\Delta \tilde{\mathbf{p}}_{ij}$、$\Delta \tilde{\mathbf{v}}_{ij}$、$\Delta \tilde{\mathbf{R}}_{ij}$：由IMU测量值计算出的预积分量（带噪声）
> - ${}_I^W \mathbf{R}_i$ 和 ${}_I^W \mathbf{R}_j$：分别为时刻 $i$ 和 $j$ 的旋转矩阵，含义是从世界坐标系 $W$ 到IMU坐标系 $I$ 的旋转。残差中的连乘顺序和取逆操作，均是为了将角度差转换到统一的坐标系下计算

---

### 第八部分：面试常见追问与回答

#### Q1：为什么要用预积分？不用行不行？

**回答**：不用预积分也可以——直接使用原始IMU测量值，在每次优化迭代时重新积分所有IMU数据。但这样**计算量巨大**：IMU频率通常是200-1000Hz，长时间运行的关键帧间包含数千个IMU测量值，每次优化迭代都重新积分不可接受。预积分的本质是把IMU测量值**压缩成一个相对运动增量**，优化时只需计算这个增量的残差和雅可比，**避免了重复积分**。

#### Q2：预积分量是在哪个坐标系下表示的？

**回答**：预积分量是在**起始时刻的IMU坐标系**下表示的，而不是在世界坐标系下。这样做的目的是使预积分量**与绝对位姿解耦**，只依赖于IMU测量值和偏置。如果预积分量在世界坐标系下表示，它就会包含起始时刻的旋转 ${}_W^I \mathbf{R}_i$，优化时每次更新 ${}_W^I \mathbf{R}_i$ 都需要重新计算，失去了预积分的意义。

#### Q3：偏置更新后，预积分量怎么办？

**回答**：当偏置发生变化时（优化迭代中），有两种处理方式：
1. **重新传播**：用新的偏置重新积分所有IMU数据——准确但计算量大
2. **一阶泰勒展开近似**：预先计算预积分量对偏置的雅可比矩阵 $\frac{\partial \Delta \mathbf{R}}{\partial \mathbf{b}_g}$、$\frac{\partial \Delta \mathbf{v}}{\partial \mathbf{b}_g}$ 等，偏置变化后直接用泰勒展开更新预积分量，**无需重新积分**

实际系统中常用方式2，因为优化迭代中偏置变化通常是微小的。

#### Q4：预积分的噪声模型怎么处理？

**回答**：预积分过程中，IMU的高斯白噪声 $\boldsymbol{\eta}_g$、$\boldsymbol{\eta}_a$ 和偏置随机游走噪声 $\boldsymbol{\eta}_{bg}$、$\boldsymbol{\eta}_{ba}$ 会传播到预积分量中。通过**线性化误差传播**可以递推计算出预积分量的协方差矩阵 $\boldsymbol{\Sigma}_{ij}$。在后端优化中，这个协方差矩阵用于**加权IMU残差**——协方差越小，残差项的权重越大。

---

### 第九部分：离散时间实现（常用中值积分）

在实际代码中，常用**中值积分法**实现预积分的递推计算：

对于第 $k$ 个IMU测量时刻到第 $k+1$ 个时刻：

$$
\begin{aligned}
\boldsymbol{\omega}_{avg} &= \frac{1}{2} \left( \boldsymbol{\omega}_{m,k} + \boldsymbol{\omega}_{m,k+1} \right) \\
\mathbf{a}_{avg} &= \frac{1}{2} \left( \mathbf{a}_{m,k} + \mathbf{a}_{m,k+1} \right)
\end{aligned}
$$

然后递推更新预积分量：

$$
\begin{aligned}
\Delta \mathbf{R}_{k+1} &= \Delta \mathbf{R}_k \exp \left( \left( \boldsymbol{\omega}_{avg} - \mathbf{b}_{g,k} \right)^\wedge \Delta t \right) \\
\Delta \mathbf{v}_{k+1} &= \Delta \mathbf{v}_k + \Delta \mathbf{R}_k \left( \mathbf{a}_{avg} - \mathbf{b}_{a,k} \right) \Delta t \\
\Delta \mathbf{p}_{k+1} &= \Delta \mathbf{p}_k + \Delta \mathbf{v}_k \Delta t + \frac{1}{2} \Delta \mathbf{R}_k \left( \mathbf{a}_{avg} - \mathbf{b}_{a,k} \right) \Delta t^2
\end{aligned}
$$

> 中值积分比欧拉积分（直接用当前时刻的测量值）精度更高，是VINS-Mono、ORB-SLAM3等主流方案中IMU预积分的主流实现方式。

---

这份推导涵盖了从IMU测量模型到预积分残差的完整链条，面试中如果被问到IMU预积分的细节，按这个逻辑讲下来会很有说服力。需要我把这份推导也整合到之前的完整面试文档中吗？😊
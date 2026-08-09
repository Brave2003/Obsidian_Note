# 滤波与状态估计

## 学习资源推荐

| 资源 | 类型 | 说明 |
|------|------|------|
| [State Estimation for Robotics - Barfoot](http://asrl.utias.utoronto.ca/~tdb/bib/barfoot_ser17.pdf) | 书籍 | 滤波器与优化统一视角，第3-4章 |
| [Probabilistic Robotics - Thrun](https://docs.ufpr.br/~danielsantos/ProbabilisticRobotics.pdf) | 书籍 | 机器人学概率方法经典，第2-3章 |
| [OpenVINS 论文](https://arxiv.org/abs/1912.10595) | 论文 | MSCKF的实际工程实现 |
| [Quaternion kinematics for the error-state Kalman filter](https://arxiv.org/abs/1711.02508) | 书籍/论文 | Sola 2017, ESKF圣经，公式推导极其清晰 |
| [The Humble Kalman Filter](https://www.kalmanfilter.net/) | 在线教程 | 从直觉到数学，适合初学KF |
| [MSCKF 原始论文 (Mourikis 2007)](https://ieeexplore.ieee.org/document/4209642) | 论文 | MSCKF开山之作 |
| [DR_CAN 卡尔曼滤波系列](https://www.bilibili.com/video/BV1dV411Y7PQ) | 视频 | B站优质中文讲解 |

---

## 核心概念详解

### 1. 卡尔曼滤波 (KF)

#### 两个步骤

**预测 (Prediction)**：
$$\hat{x}_{k|k-1} = F_k \hat{x}_{k-1} + B_k u_k$$
$$P_{k|k-1} = F_k P_{k-1} F_k^T + Q_k$$

**更新 (Update)**：
$$K_k = P_{k|k-1} H_k^T (H_k P_{k|k-1} H_k^T + R_k)^{-1}$$
$$\hat{x}_k = \hat{x}_{k|k-1} + K_k (z_k - H_k \hat{x}_{k|k-1})$$
$$P_k = (I - K_k H_k) P_{k|k-1}$$

**通俗理解**：
- 预测 = 根据运动模型，"猜"下一时刻状态在哪里
- 更新 = 用测量值"纠正"猜测
- 卡尔曼增益 $K$ = 更相信预测(小K)还是更相信测量(大K)

### 2. 扩展卡尔曼滤波 (EKF)

非线性系统：$x_k = f(x_{k-1}, u_k) + w_k$, $z_k = h(x_k) + v_k$

**EKF的核心**：对非线性函数做一阶泰勒展开，然后套用KF公式。

$$F_k = \left.\frac{\partial f}{\partial x}\right|_{\hat{x}_{k-1}}, \quad H_k = \left.\frac{\partial h}{\partial x}\right|_{\hat{x}_{k|k-1}}$$

**EKF在SLAM中的问题**：状态量很大时，协方差矩阵 $P$ 是 $N\times N$，更新计算量 $O(N^2)$。

### 3. 误差状态卡尔曼滤波 (ESKF)

**核心思想**：不直接估计状态（如四元数有约束），而是估计"误差状态"。

- **名义状态 (Nominal State)**：按运动方程直接积分，可能有累积误差
- **误差状态 (Error State)**：用KF估计名义状态的误差，再注入校正

$$\tilde{x} = x \ominus \hat{x}$$

**为什么用误差状态**：
1. 误差状态通常很小 → 线性近似更准确
2. 误差状态在0附近 → 无奇异
3. 可以用标准KF公式 (不需要处理旋转约束)

### 4. MSCKF (Multi-State Constraint Kalman Filter)

**MSCKF的核心创新**：不是把特征点加入状态向量，而是用多帧相机之间的**对极约束**来更新状态。

**工作流程**：
1. 维护一个**滑动窗口**内最近N帧的相机位姿
2. 当某个特征点被多帧观测到 → 产生约束
3. 三角化该特征点 → 计算重投影误差
4. 将该误差投影到**特征点的零空间**（消除特征点位置，只留位姿约束）
5. 用压缩后的残差更新滤波器

**优点**：
- 状态量只有位姿，**不需要维护地图点**
- 计算复杂度 $O(N^3)$，其中 $N$ 是窗口大小（固定值）

### 5. 可观性与退化

**可观性**：能否从测量值唯一确定系统状态。

**VIO中的问题**：
- 悬停/匀速直线运动：加速度测量只有重力 + 偏置 → 尺度不可观
- 纯旋转：没有平移视差 → 深度不可观

**OpenVINS 为什么强调可观性**：
- EKF的线性化点不同会导致系统维度的"虚假可观"
- 需要正确选择线性化点保证一致性 (FEJ, First-Estimate Jacobian)

### 6. 滤波法 vs 优化法

| | 滤波法 (Filtering) | 优化法 (Optimization) |
|---|---|---|
| 代表 | EKF, MSCKF, OpenVINS | VINS-Fusion, ORB-SLAM |
| 思路 | 递推，只维护当前状态 | 批量优化窗口内所有状态 |
| 计算量 | 固定（状态维度的三次方） | 随窗口大小增长 |
| 效率 | 更快，适合实时性高场景 | 更准，可以反复线性化 |
| 信息利用 | 边缘化旧信息 | 保留更多历史约束 |
| 难度 | 理论门槛高（可观性、一致性） | 工程复杂（滑窗、边缘化管理） |

**实际趋势**：现代SLAM系统中两种方法在融合——用优化做局部精调，用边缘化传递先验信息。

---

## 动手实践清单

- [ ] 手动实现一个1D卡尔曼滤波器（位置+速度估计）
- [ ] 用EKF实现2D机器人定位（已知路标）
- [ ] 阅读OpenVINS中 `Propagator.cpp` 的IMU传播
- [ ] 阅读OpenVINS中 `UpdaterHelper.cpp` 的观测更新
- [ ] 对比EKF和滑动窗口优化在VIO中的轨迹差异
- [ ] 分析MSCKF为什么比EKF-SLAM效率高
- [ ] 推导VINS-Mono的IMU预积分误差传播

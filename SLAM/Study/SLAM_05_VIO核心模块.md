# VIO 核心模块

## 学习资源推荐

| 资源 | 类型 | 说明 |
|------|------|------|
| [VINS-Mono 论文](https://arxiv.org/abs/1708.03852) | 论文 | 最经典的优化式VIO，必读 |
| [IMU Preintegration (Forster 2015/2017)](https://ieeexplore.ieee.org/document/7557075) | 论文 | IMU预积分理论，RSS 2015 + IJRR 2017 |
| [VINS-Mono 代码注释版](https://github.com/HeYijia/vins_course) | 代码 | 知乎贺一家维护的中文注释版 |
| [崔华坤 - VINS-Mono论文/代码精讲](https://www.bilibili.com/video/BV1hC411o7en) | 视频 | 逐行代码讲解 |
| [OpenVINS 官方文档](https://docs.openvins.com/) | 文档 | 滤波式VIO完整文档 |
| [VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion) | 代码 | 双目/双目+IMU/双目+GPS多版本 |
| [深蓝学院 VIO课程](https://www.shenlanxueyuan.com/course/430) | 课程 | 付费课程，质量高 |
| [Kimera-VIO 论文](https://arxiv.org/abs/1910.02490) | 论文 | 语义VIO系统 |

---

## 核心概念详解

### 1. IMU预积分

**为什么要预积？**

普通积分：两帧之间的IMU积分依赖起始位姿，如果起始位姿在优化中被调整了，所有的积分都要重算。

**预积分的核心思想**：将IMU测量积分成**相对于起始帧的增量**，这个增量不依赖起始位姿：

$$\Delta R_{ij} = \prod_{k=i}^{j-1} \exp\left((\tilde{\omega}_k - b_{g,k})\Delta t\right)$$
$$\Delta v_{ij} = \sum_{k=i}^{j-1} \Delta R_{ik} (\tilde{a}_k - b_{a,k}) \Delta t$$
$$\Delta p_{ij} = \sum_{k=i}^{j-1} \left[\Delta v_{ik} \Delta t + \frac{1}{2} \Delta R_{ik} (\tilde{a}_k - b_{a,k}) \Delta t^2\right]$$

当位姿被调整时，只需要重新组合：
$$R_j = R_i \cdot \Delta R_{ij}$$

**偏置修正**：偏置变化时，预积分量需要修正。Forster 论文中使用一阶近似：
$$\Delta R_{ij}(b_g + \delta b_g) \approx \Delta R_{ij}(b_g) \cdot \exp\left(\frac{\partial \Delta R_{ij}}{\partial b_g} \delta b_g\right)$$

### 2. VIO 初始化

初始化的核心是恢复：**重力方向、速度、尺度、IMU偏置**。

**VINS-Mono 初始化步骤**：

1. **纯视觉SfM**：用多帧图像做Structure from Motion，得到无尺度(up-to-scale)的位姿
2. **视觉-IMU对齐**：
   - 利用视觉帧间的相对旋转，标定陀螺仪偏置 $b_g$
   - 确定重力方向矢量
   - 恢复尺度 $s$ (单目VIO的关键)
   - 恢复每个关键帧的速度

**为什么初始化困难？**
- 需要足够的视差(不能纯旋转)
- 加速度激励不足时尺度不可观
- 偏置变化快时需要快速初始化

**初始化的输出**：重力的方向+大小、尺度因子、初始速度、IMU偏置初值。

### 3. 紧耦合 vs 松耦合

| | 松耦合 (Loose) | 紧耦合 (Tight) |
|---|---|---|
| 方式 | 视觉和IMU各自计算，结果再融合 | 视觉和IMU原始测量联合优化 |
| 视觉残差 | 位姿误差 | 像素重投影误差 |
| IMU残差 | 位姿/速度误差 | IMU预积分残差 |
| 精度 | 损失信息 | 最优 |
| 实现难度 | 简单 | 复杂 |
| 代表 | 早期系统 | VINS-Mono, ORB-SLAM3 |

**实际中都用紧耦合**，松耦合只在快速原型或多传感器校验时使用。

### 4. 滑动窗口优化

VINS-Mono 维护一个10帧左右的关键帧窗口：

**优化变量**：
- 滑动窗口内每个关键帧的 $p, v, q, b_a, b_g$
- 窗口外的历史信息通过**边缘化**变成先验约束

**约束来源**：
- 视觉：特征点的重投影残差
- IMU：预积分残差
- 回环：回环帧之间的相对位姿约束
- 先验：边缘化产生的先验约束

**非线性优化**：Ceres Solver，LM方法

### 5. 边缘化

当窗口内帧数达到上限，需要丢弃最老或次新的关键帧：

**丢弃策略**：
- 如果次新帧是关键帧 → 丢弃最老帧
- 如果次新帧不是关键帧 → 丢弃次新帧（保留更多信息）

**舒尔补边缘化**：
1. 将状态分为要保留的 $x_m$ 和要丢弃的 $x_r$
2. 对信息矩阵进行舒尔补：$\Lambda_{new} = \Lambda_{mm} - \Lambda_{mr} \Lambda_{rr}^{-1} \Lambda_{rm}$
3. 先验残差：$b_{new} = b_m - \Lambda_{mr} \Lambda_{rr}^{-1} b_r$

**边缘化的代价**：信息矩阵变稠密 (fill-in)，产生"蛛网"结构。

### 6. 在线外参与时间偏移

**在线外参估计**：在VIO优化中把相机-IMU外参 $T_{c}^{i}$ 作为优化变量。

**时间偏移估计**：
- 实际中相机和IMU的时间戳可能不对齐
- 建模为时间差 $t_d$：将IMU在时间 $t + t_d$ 的位姿用于视觉测量
- 在优化中估计 $t_d$ 或使用互相关方法

### 7. 评测指标

**ATE (Absolute Trajectory Error)**：估计轨迹与真值轨迹的对齐后绝对误差
$$\text{ATE} = \sqrt{\frac{1}{N} \sum_{i} \|p_{est,i} - p_{gt,i}\|^2}$$

**RPE (Relative Pose Error)**：相邻帧之间的相对位姿误差
$$\text{RPE} = \sqrt{\frac{1}{N-\Delta} \sum_{i} \|(p_{est,i+\Delta} - p_{est,i}) - (p_{gt,i+\Delta} - p_{gt,i})\|^2}$$

**区别**：ATE看全局一致性，RPE看局部漂移。VIO通常关注RPE，回环后关注ATE。

---

## 动手实践清单

- [ ] 跑通 VINS-Mono 在 EuRoC 上的示例
- [ ] 用 GDB 完整跟踪一次预积分的计算
- [ ] 理解初始化代码中重力恢复的过程
- [ ] 修改 VINS 的关键帧选择策略，观察影响
- [ ] 关闭/开启回环，对比 ATE 和 RPE
- [ ] 画出滑动窗口的因子图，标注每种残差的雅可比维度
- [ ] 阅读边缘化的舒尔补代码
- [ ] 用 evo 工具计算并可视化 ATE 和 RPE

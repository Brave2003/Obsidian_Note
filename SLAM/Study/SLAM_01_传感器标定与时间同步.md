# 传感器、标定与时间同步

## 学习资源推荐

### 相机模型与标定

| 资源                                                                                                 | 类型   | 说明                         |
| -------------------------------------------------------------------------------------------------- | ---- | -------------------------- |
| [Camera Calibration - OpenCV官方教程](https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html) | 官方文档 | 张正友标定法完整实现                 |
| [Multiple View Geometry (MVG) 第6-7章](https://www.robots.ox.ac.uk/~vgg/hzbook/)                     | 书籍   | 相机模型权威参考，必读                |
| [Kalibr 官方文档](https://github.com/ethz-asl/kalibr/wiki)                                             | 开源工具 | 相机-IMU标定的工业级工具             |
| [鱼眼相机模型详解](https://docs.opencv.org/4.x/db/d58/group__calib3d__fisheye.html)                        | 官方文档 | OpenCV鱼眼模型(Kannala-Brandt) |

### 相机-IMU标定

| 资源 | 类型 | 说明 |
|------|------|------|
| [Kalibr Wiki - 多相机-IMU标定](https://github.com/ethz-asl/kalibr/wiki/camimu-calibration) | Wiki | 手把手标定流程 |
| VINS-Mono 标定工具 | 代码 | `vins_estimator` 中的在线标定 |
| [AprilTag](https://april.eecs.umich.edu/software/apriltag) | 工具 | 用于制作标定板，替代棋盘格 |

### LiDAR-IMU标定

| 资源 | 类型 | 说明 |
|------|------|------|
| [LI-Init](https://github.com/hku-mars/LI-Init) | 开源 | 港大Mars实验室，LiDAR-IMU初始化 |
| [LI-Calib](https://github.com/Unsigned-Long/LiDAR-IMU-Calibration) | 开源 | 连续时间LiDAR-IMU标定 |
| [APRIL-Calib](https://github.com/versatran01/apriltag-calibration) | 开源 | 利用AprilTag做多传感器标定 |

### 时间同步

| 资源 | 类型 | 说明 |
|------|------|------|
| [时间同步综述](https://arxiv.org/abs/2109.04979) | 论文 | 多传感器时间同步方法综述 |
| [OpenVINS 时间偏移估计](https://docs.openvins.com/update-delay.html) | 文档 | 在线估计时间偏移的原理与实现 |

### IMU模型

| 资源 | 类型 | 说明 |
|------|------|------|
| [IMU Noise Model](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model) | Wiki | Kalibr对IMU噪声模型的解释 |
| [Allan Variance 教程](https://mwrona.com/posts/gyroscope-allan-variance/) | 博客 | Allan方差图读法图解 |
| [IMU Preintegration 原始论文 (Forster 2015)](https://arxiv.org/abs/1512.02363) | 论文 | RSS 2015, IMU预积分开山之作 |

---

## 核心概念详解

### 1. 相机模型

#### 针孔相机模型

世界坐标 $(X_w, Y_w, Z_w)$ 到像素 $(u, v)$ 的投影：

$$
s \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} = 
\begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}
\begin{bmatrix} R & t \end{bmatrix}
\begin{bmatrix} X_w \\ Y_w \\ Z_w \\ 1 \end{bmatrix}
$$

其中 $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$ 为内参矩阵。

#### 畸变模型

**径向畸变 (Radial)**：
$$x_{distorted} = x(1 + k_1 r^2 + k_2 r^4 + k_3 r^6)$$

**切向畸变 (Tangential)**：
$$x_{distorted} = x + [2p_1 xy + p_2(r^2 + 2x^2)]$$

**鱼眼模型 (Kannala-Brandt)**：
$$\theta = \arctan(r), \quad r_d = \theta(1 + k_1\theta^2 + k_2\theta^4 + k_3\theta^6 + k_4\theta^8)$$

#### 面试常见问题

1. **为什么需要标定？** 镜头制造公差、组装误差导致实际投影偏离理想针孔模型
2. **重投影误差的含义？** 标定板角点的3D位置用估计参数投影回2D，与检测位置的像素差
3. **标定板移动要注意什么？** 覆盖图像四角和中心、不同角度、不同距离；避免纯旋转
4. **双目标定的核心目标？** 获得两个相机之间的 $R, t$ 和各自内参

### 2. 相机-IMU外参标定

核心问题是估计 $T_{c}^{i}$ (从IMU系到相机系的变换) 和时间偏移 $t_d$。

**手眼标定问题**：已知两个传感器各自运动的相对变换，求解它们之间的固定外参。
$$AX = XB$$

- $A$: 相机在连续两帧之间的相对运动
- $B$: IMU在连续两帧之间的相对运动
- $X$: 待求的相机-IMU外参 $T_{c}^{i}$

**Kalibr** 使用连续时间样条曲线 + 角点重投影来求解。

### 3. LiDAR运动畸变

LiDAR扫描过程中载体在运动，导致一帧内的点云不是在同一时刻采集的。

**去畸变方法**：
- **匀速模型**：假设帧内匀速，用起止位姿线性插值
- **IMU辅助**：利用IMU高频输出(100-400Hz)推算帧内运动
- **连续时间优化**：将轨迹建模为B样条，在优化中求解

### 4. IMU测量模型

陀螺仪和加速度计的测量包含：
- 真值信号
- 缓慢变化的偏置 $b$
- 高斯白噪声 $n$

$$\tilde{\omega} = \omega + b_g + n_g$$
$$\tilde{a} = a + b_a + n_a + g$$

偏置建模为随机游走：$\dot{b} = n_b$

**Allan方差**：用来分析不同频率下噪声的特性：
- 斜率 -1/2 → 角度随机游走(白噪声)
- 斜率 0 → 偏置不稳定性
- 斜率 +1/2 → 速率随机游走

### 5. 时间同步

**硬件同步**：用同一时钟信号触发所有传感器（PPS/GPS）
**软件同步**：利用时间戳插值或在线估计偏移

**关键问题**：不同传感器的数据到达时间不同，如果直接按时间戳配对会产生误差。例如100ms的时间偏移在10m/s的速度下会产生1m的位置误差。

---

## 动手实践清单

- [ ] 用 OpenCV 完成一次相机标定，打印内参和畸变系数
- [ ] 将 LiDAR 点云投影到图像上，检查标定结果
- [ ] 用 Kalibr 录制并标定一个相机-IMU系统
- [ ] 阅读 OpenVINS 中 IMU 传播的代码 (`src/core/Propagator.cpp`)
- [ ] 计算一个IMU的噪声密度和随机游走参数
- [ ] 画一张传感器时序图，标注各环节延迟

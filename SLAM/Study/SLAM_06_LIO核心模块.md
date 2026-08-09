# LIO 核心模块

## 学习资源推荐

| 资源 | 类型 | 说明 |
|------|------|------|
| [LOAM 论文](https://www.ri.cmu.edu/pub_files/2014/7/Ji_LidarMapping_RSS2014_v8.pdf) | 论文 | LiDAR里程计开山之作 |
| [LeGO-LOAM 论文](https://arxiv.org/abs/1902.07022) | 论文 | 轻量级地面优化LOAM |
| [FAST-LIO2 论文](https://arxiv.org/abs/2107.06829) | 论文 | 当前最快最鲁棒的开源LIO |
| [LIO-SAM 论文](https://arxiv.org/abs/2107.04729) | 论文 | 基于因子图的紧耦合LIO |
| [FAST-LIO2 源码](https://github.com/hku-mars/FAST-LIO2) | 代码 | 港大Mars实验室 |
| [LIO-SAM 源码](https://github.com/TixiaoShan/LIO-SAM) | 代码 | 因子图LIO |
| [PCL 官方教程](https://pcl.readthedocs.io/projects/tutorials/en/latest/) | 文档 | 点云库PCL完整教程 |
| [KISS-ICP](https://github.com/PRBonn/kiss-icp) | 代码 | 极简ICP，适合学习 |

---

## 核心概念详解

### 1. 点云去畸变

LiDAR扫描一帧需要100ms，而载体在这100ms内可能已经移动了几米。

**经典去畸变方法 (LOAM方式)**：
1. 假设帧内匀速运动
2. 利用IMU估计每时刻的位姿
3. 用线性插值将每个点变换到帧结束时刻的坐标系

**公式**：
$$T_{k,i} = \frac{t_i - t_k}{t_{k+1} - t_k} \cdot T_{k+1,k}$$
$$P_{corrected} = T_{k,i} \cdot P_{raw}$$

**FAST-LIO2 的方法 (Backward Propagation)**：
- 利用IMU高频特性，反向传播每个点
- 处理更精确，不依赖匀速假设

### 2. Scan-to-Scan vs Scan-to-Map

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Scan-to-Scan | 相邻两帧点云匹配 | 计算快 | 漂移快 |
| Scan-to-Map | 当前帧与累积地图匹配 | 精度高 | 计算量大，地图质量影响大 |

**实际用法**：
- LOAM: scan-to-scan 给初始位姿，scan-to-map 精化
- FAST-LIO2: 直接 scan-to-map (因为地图是增量kd-tree，查询很快)

### 3. 点面/点线残差

**点到点的 ICP**：
$$e = \|T \cdot p - q\|$$

**点到线的残差** (常用于边缘特征)：
$$e_{line} = \|(T p - q_1) \times (T p - q_2)\|$$

即当前点 $p$ 到由 $q_1, q_2$ 定义的直线的距离。

**点到面的残差** (常用于平面特征)：
$$e_{plane} = (T p - q_0)^T n$$

即当前点 $p$ 到由 $q_0$ 和法向量 $n$ 定义的平面的距离。

**为什么点到线/面比点到点好？**
LiDAR点云是稀疏采样，不同帧的激光点**大概率不落在同一物理点上**。但边缘上的点会落在同一条线上，平面上的点会落在同一个面上。

### 4. ICP 变体

| 变体 | 特点 | 何时使用 |
|------|------|----------|
| Point-to-Point | 经典ICP | 密集点云 |
| Point-to-Plane | 收敛更快 | 平面丰富的环境 (室内) |
| GICP | 泛化ICP，考虑局部协方差 | 非结构化环境 |
| NDT | 正态分布变换 | 大尺度雷达 |
| KISS-ICP | 极简实现 + 自适应阈值 | 学习用途 |

### 5. FAST-LIO2 的核心创新

1. **迭代卡尔曼滤波 (IESKF)**：观测在迭代中重复线性化
2. **增量 kd-tree (ikd-Tree)**：支持动态插入/删除的地图数据结构
3. **反向传播**：利用IMU数据去畸变，精度更高
4. **紧耦合**：IMU预积分与LiDAR残差联合，比LOAM更鲁棒

**FAST-LIO2 vs LIO-SAM 对比**：

| | FAST-LIO2 | LIO-SAM |
|---|---|---|
| 核心方法 | IESKF | 因子图 (GTSAM) |
| 地图结构 | ikd-Tree (增量) | 体素网格 |
| 速度 | 极快 (1000+ Hz点处理) | 中速 |
| GPU需求 | 不需要 | 不需要 |
| 回环 | 无内置 | 有 (Scan Context) |
| 多传感器 | IMU + LiDAR | IMU + LiDAR + GPS |

### 6. 退化问题

**退化场景**：环境中几何约束不足：
- **长廊/隧道**：只有两个面的约束，沿着隧道方向的平移不可观
- **空旷场地**：地平面提供俯仰/滚转/高度约束，但水平平移和偏航约束弱
- **对称环境**：产生多个局部极值

**检测方法**：
- 对约束矩阵做特征值分解
- 小特征值对应的方向 = 退化方向

**缓解策略**：
- 退化方向上更依赖IMU
- 添加运动先验约束

### 7. 实时性优化

- **特征提取**：计算曲率，提取边缘和平面点，降低数据量
- **空间下采样**：体素滤波，避免密集区域占主导
- **数据结构**：kd-tree加速最近邻搜索
- **增量处理**：新数据直接插入地图（FAST-LIO2的ikd-tree）

---

## 动手实践清单

- [ ] 跑通 FAST-LIO2 在 M2DGR/LVI-SAM 数据集上
- [ ] 手写一个简单的点到面ICP配准
- [ ] 理解LOAM中边缘/平面特征提取的代码
- [ ] 阅读 ikd-Tree 的插入和删除实现
- [ ] 对比有无去畸变的点云地图质量
- [ ] 画出 LIO-SAM 的因子图结构
- [ ] 测量不同分辨率下的配准精度和速度

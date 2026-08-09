# 主流 SLAM 框架学习路线

## 框架选择与学习顺序

```
第一梯队 (深读+修改)      第二梯队 (阅读+了解)
OpenVINS 或 VINS-Fusion    FAST-LIO2 或 LIO-SAM
ORB-SLAM3 (完整阅读)       DSO (按需)
```

---

## 1. OpenVINS — 滤波式 VIO

### 推荐理由
- 代码质量高，文档齐全 (docs.openvins.com)
- 是学习 MSCKF、EKF、误差状态和可观性的最佳载体
- 每一帧的处理都在一个完整的滤波器循环中

### 学习路线 (2-3周)

**第一周：跑通 + 理解数据流**
```bash
git clone https://github.com/rpng/open_vins
# 跑 EuRoC 数据集
roslaunch ov_msckf subscribe.launch config:=euroc_mav
```

**第二周：跟踪一次完整滤波循环**
- 入口：`VioManager::feed_measurement_imu()`
- IMU传播：`Propagator::propagate()`
- 相机测量更新：`UpdaterMSCKF::update()`
- 状态增广：`StateHelper::augment_clone()`

**第三周：修改实验**
- 修改关键帧选择策略
- 添加/移除某种约束
- 对比有/无FEJ的轨迹差异

### 核心模块
| 模块 | 文件 | 要理解的点 |
|------|------|-----------|
| 状态量 | `State.h` | 有哪些状态、如何增广/边缘化 |
| IMU传播 | `Propagator.cpp` | 均值传播、协方差传播 |
| MSCKF更新 | `UpdaterMSCKF.cpp` | 特征三角化、零空间投影 |
| FEJ | `UpdaterHelper.cpp` | 为什么需要FEJ |
| 初始化 | `InertialInitializer.cpp` | 静止初始化 |

### 面试高频
- MSCKF 和 EKF-SLAM 的区别？为什么MSCKF更高效？
- FEJ 解决了什么问题？
- 可观性为什么重要？什么场景会退化？

---

## 2. VINS-Fusion — 优化式 VIO

### 推荐理由
- 中文社区资源最丰富
- 覆盖完整VIO管线：初始化、滑窗、边缘化、回环
- ROS工程化程度高，接近工业可用的系统
- 支持双目、GPS多传感器扩展

### 学习路线 (3周)

**第一周：跑通 + 理解初始化**
```bash
git clone https://github.com/HKUST-Aerial-Robotics/VINS-Fusion
# 跑 EuRoC 数据集
roslaunch vins vins_rviz.launch
rosrun vins vins_node /PATH_TO_CONFIG/euroc/euroc_stereo_imu_config.yaml
```

**第二周：跟踪一次完整优化**
- 特征跟踪：`feature_tracker` 节点
- IMU预积分：`imu_factor.h` 中的 `IntegrationBase`
- 滑窗优化：`estimator.cpp` 中的 `optimization()`
- 边缘化：`estimator.cpp` 中的 `marginalize()`

**第三周：修改实验**
- 修改边缘化策略
- 改变关键帧条件
- 添加自定义约束

### 核心文件
| 文件 | 内容 | 重点 |
|------|------|------|
| `estimator.cpp` | 状态管理、优化、边缘化 | 整个系统的核心 |
| `estimator_node.cpp` | ROS节点入口 | 理解数据流 |
| `feature_tracker.cpp` | 视觉前端 | 光流跟踪 |
| `factors/imu_factor.h` | IMU预积分因子 | 残差+雅可比 |
| `initial/initial_alignment.cpp` | 视觉IMU对齐 | 初始化核心 |
| `loop_closing/` | 回环检测与修正 | 位姿图优化 |

### 面试高频
- 边缘化的策略：丢弃最老帧还是次新帧？为什么？
- 初始化失败的场景有哪些？
- 重定位和回环修正有什么区别？

---

## 3. ORB-SLAM3 — 完整SLAM系统

### 推荐理由
- 功能最全(单目/双目/RGB-D/IMU、多地图、回环)
- 工业级代码，理解完整SLAM系统的架构
- 经典的3线程+Atlas架构

### 学习路线 (2-3周，可以并行)

**聚焦阅读3条线程**：
1. **Tracking** — 当前帧的实时位姿估计 (30Hz)
2. **LocalMapping** — 关键帧和地图点的局部优化 (后台)
3. **LoopClosing** — 回环检测 + 全局位姿图优化 + 全局BA (后台)

### 关键代码
| 文件 | 作用 |
|------|------|
| `Tracking.cc` | 跟踪线程：ORB提取、运动模型、参考关键帧跟踪、重定位 |
| `LocalMapping.cc` | 局部建图：新关键帧处理、地图点创建/剔除、局部BA |
| `LoopClosing.cc` | 回环检测：词袋查询、Sim3计算、位姿图优化、全局BA |
| `Atlas.h/cc` | 多地图管理：活动地图切换、地图合并 |
| `ORBmatcher.cc` | ORB特征匹配：投影匹配、词袋匹配 |

### 面试高频
- 三条线程之间的关系？LocalMapping慢了会影响Tracking吗？
- 关键帧的选择条件？
- 地图点的创建和剔除条件？

---

## 4. DSO — 直接法

### 推荐理由
- 理解直接法的最佳载体
- 理解了DSO就能理解"为什么ORB-SLAM用特征法"

### 只需理解
- 光度误差的定义和雅可比
- 滑动窗口优化的边缘化策略(Marginalization)
- 为什么直接法对曝光敏感但更适合弱纹理

### 不需要
- 逐行阅读全部代码
- 跑全量实验
- 修改核心模块（除非做直接法方向）

---

## 5. FAST-LIO2 — 高实时LIO

### 推荐理由
- 状态估计思想(MSCKF→IESKF的进化)
- 增量数据结构(ikd-tree)的设计
- 工程极致优化(1000+ Hz点处理)

### 核心理解
- IESKF：迭代中重新线性化观测模型
- ikd-Tree：如何支持增量插入/删除
- 如何保证实时性（多线程、数据结构选择）

---

## 6. LIO-SAM — 因子图LIO

### 推荐理由
- 理解因子图优化的实际应用
- 学习GTSAM的使用方式
- 理解多因子(IMU预积分 + LiDAR里程计 + GPS + 回环)的融合

### 核心理解
- GTSAM的因子图构建
- IMU预积分因子的使用
- Scan Context回环检测

---

## 面试答案模板

**Q: "你深入研究了哪个SLAM框架？"**

```
1. [框架名称]：VINS-Fusion
2. [研究深度]：
   - 完整跟踪了从IMU数据到VIO输出的全流程
   - 理解了初始化、预积分、滑窗优化和边缘化
   - 修改了关键帧选择策略并做了对比实验
3. [遇到的困难]：
   - 初始化失败时发现是因为加速度激励不足
   - 通过分析协方差矩阵确认尺度不可观
4. [具体改进]：
   - 修改了XXXX → ATE从X降到Y
```

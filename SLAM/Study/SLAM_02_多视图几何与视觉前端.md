# 多视图几何与视觉前端

## 学习资源推荐

### 核心书籍与课程

| 资源 | 类型 | 说明 |
|------|------|------|
| [Multiple View Geometry in Computer Vision (MVG)](https://www.robots.ox.ac.uk/~vgg/hzbook/) | 书籍 | 多视图几何圣经，Part I+II必读 |
| [视觉SLAM十四讲 - 高翔](https://github.com/gaoxiang12/slambook2) | 书籍+代码 | 中文SLAM入门经典，第5-9讲 |
| [CVPR 2017 多视图几何教程](https://www.cvl.isy.liu.se/education/graduate/mvg2017/) | 课程 | 瑞典林雪平大学，有PPT和习题 |
| [First Principles of Computer Vision - Shree Nayar](https://fpcv.cs.columbia.edu/) | 视频课 | 哥伦比亚大学，YouTube有全套 |

### 特征提取与匹配

| 资源 | 类型 | 说明 |
|------|------|------|
| [ORB-SLAM2 论文](https://arxiv.org/abs/1610.06475) | 论文 | ORB特征在SLAM中的实际应用 |
| [SuperPoint 论文](https://arxiv.org/abs/1712.07629) | 论文 | 自监督学习特征点 |
| [SuperGlue 论文](https://arxiv.org/abs/1911.11763) | 论文 | 基于GNN的特征匹配 |
| [LightGlue 论文](https://arxiv.org/abs/2306.13643) | 论文 | 轻量级注意力匹配，比SuperGlue更快 |
| [OpenCV Feature Matching Tutorial](https://docs.opencv.org/4.x/dc/dc3/tutorial_py_matcher.html) | 官方文档 | 动手学习特征匹配 |

### 直接法与光流

| 资源 | 类型 | 说明 |
|------|------|------|
| [DSO 论文](https://arxiv.org/abs/1607.02565) | 论文 | 直接稀疏里程计 |
| [Lucas-Kanade 20 Years On](https://www.ri.cmu.edu/pub_files/pub3/baker_simon_2004_1/baker_simon_2004_1.pdf) | 综述 | LK光流和直接法的理论基础 |
| [RAFT 光流](https://arxiv.org/abs/2003.12039) | 论文 | 深度学习光流SOTA |

---

## 核心概念详解

### 1. 坐标系与投影

世界系 → 相机系 → 归一化平面 → 像素平面：

$$P_c = R_{cw} P_w + t_{cw}$$
$$\begin{bmatrix} u \\ v \end{bmatrix} = \begin{bmatrix} f_x \frac{X_c}{Z_c} + c_x \\ f_y \frac{Y_c}{Z_c} + c_y \end{bmatrix}$$

### 2. 对极几何

**核心公式**：对于一对匹配点 $p_1, p_2$：

$$p_2^T F p_1 = 0$$
$$p_2^T K^{-T} t^\wedge R K^{-1} p_1 = 0$$

其中：
- $E = t^\wedge R$ 是本质矩阵 (Essential Matrix)
- $F = K^{-T} E K^{-1}$ 是基础矩阵 (Fundamental Matrix)

**八点法**：8对匹配点求解 $E$，再SVD分解得到 $R$ 和 $t$。

**关键理解**：$E$ 只编码两个相机之间的相对旋转和平移($R$, $t$)，不涉及内参。$F$ 则包含内参。

### 3. PnP (Perspective-n-Point)

已知n个3D点及其对应的2D投影，估计相机位姿。

**方法分类**：
- **P3P**：最少3个点（需要额外1个点验证），有4个解
- **EPnP**：高效方法，时间复杂度 O(n)，OpenCV默认
- **UPnP**：可同时估计外参和内参
- **BA优化**：将PnP作为非线性优化问题

**RANSAC**：Random Sample Consensus
- 随机采样最小子集（如8对点求解F矩阵）
- 计算模型
- 统计内点数量
- 重复多次，选内点最多的模型

### 4. 特征法 vs 直接法 vs 光流法

| 方法 | 优点 | 缺点 | 代表系统 |
|------|------|------|----------|
| 特征法 | 对光照鲁棒、可做回环 | 弱纹理区失效、信息利用少 | ORB-SLAM |
| 直接法 | 利用全部像素、适合弱纹理 | 光度假设强、易受曝光影响 | DSO |
| 光流法 | 速度快、密集对应 | 长基线差、纯旋转失效 | SVO |

### 5. 关键帧管理

**为什么需要关键帧？**
- 减少优化变量数量
- 保证帧之间有足够视差
- 避免信息冗余

**关键帧选择策略**（以 ORB-SLAM 为例）：
1. 距上一关键帧经过20帧以上
2. 当前帧跟踪到的地图点少于一定数量
3. 距上一关键帧的共视程度低于一定比例

### 6. 常见困难场景

| 场景 | 问题 | 缓解方法 |
|------|------|----------|
| 低纹理 | 特征点太少 | 直接法、线面特征 |
| 重复纹理 | 误匹配多 | 几何验证、2D-2D一致性 |
| 运动模糊 | 特征质量下降 | IMU预测、提高帧率 |
| 光照变化 | 光度一致性被破坏 | 特征法替代直接法 |
| 动态物体 | 外点污染 | RANSAC、语义剔除 |

---

## 动手实践清单

- [ ] 实现8点法求本质矩阵，并用SVD恢复 $R, t$
- [ ] 用OpenCV实现三角化 (triangulatePoints)
- [ ] 实现PnP + RANSAC的位姿估计
- [ ] 用ORB特征匹配两张图像，画对极线
- [ ] 阅读 ORB-SLAM3 的 Tracking 线程源码
- [ ] 可视化关键帧、地图点和共视图
- [ ] 对比不同特征方法(ORB vs SIFT vs SuperPoint)在你的数据集上的表现

---

## 关键公式速查

**对极约束**：$x_2^T E x_1 = 0$ (归一化坐标)，$p_2^T F p_1 = 0$ (像素坐标)

**三角化**：已知 $P_1, P_2$ 为投影矩阵，解 $x = PX$:
$$\begin{bmatrix} [p_1]_\times P_1 \\ [p_2]_\times P_2 \end{bmatrix} X = 0$$

**重投影误差**：$\sum_{i} \sum_{j} \|p_{ij} - \pi(P_i, X_j)\|^2$

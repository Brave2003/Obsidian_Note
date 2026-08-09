# 深度学习与 BEV 端到端自动驾驶模型前端

## 学习资源推荐

### 深度学习基础 (只学到够用即可)

| 资源 | 类型 | 说明 |
|------|------|------|
| [动手学深度学习 - 李沐 (d2l.ai)](https://d2l.ai/) | 在线书+视频 | 中文最佳DL入门，只看CNN/TF/Attention章节 |
| [PyTorch 官方教程](https://pytorch.org/tutorials/) | 官方文档 | 60min blitz 入门，再学数据加载和训练循环 |
| [The Annotated Transformer](http://nlp.seas.harvard.edu/annotated-transformer/) | 博客 | Transformer论文的逐行注释实现 |
| [3D Bounding Box Estimation Using Deep Learning](https://arxiv.org/abs/1612.00496) | 论文 | 入门3D检测 |

### BEV 感知 (重点)

| 资源 | 类型 | 说明 |
|------|------|------|
| [Lift-Splat-Shoot (LSS) 论文](https://arxiv.org/abs/2008.05711) | 论文 | BEV特征构建的开创性工作，必读 |
| [BEVDet 论文](https://arxiv.org/abs/2112.11790) | 论文 | 高效的BEV感知基线 |
| [BEVDet4D 论文](https://arxiv.org/abs/2203.17054) | 论文 | 加入时序的BEVDet，项目首选基线 |
| [BEVFormer 论文](https://arxiv.org/abs/2203.17270) | 论文 | 基于Transformer Deformable Attention的BEV |
| [BEVFormer 中文解读](https://zhuanlan.zhihu.com/p/508172899) | 博客 | 知乎详细解读 |
| [BEVFusion 论文](https://arxiv.org/abs/2205.13542) | 论文 | 多传感器BEV融合 |
| [MMDetection3D](https://github.com/open-mmlab/mmdetection3d) | 代码 | OpenMMLab的3D检测框架 |

### 端到端自动驾驶

| 资源 | 类型 | 说明 |
|------|------|------|
| [UniAD 论文](https://arxiv.org/abs/2212.10156) | 论文 | 统一自动驾驶框架，理解输入输出接口 |
| [VAD 论文](https://arxiv.org/abs/2303.14219) | 论文 | 向量化自动驾驶 |

### nuScenes

| 资源 | 类型 | 说明 |
|------|------|------|
| [nuScenes 官方教程](https://www.nuscenes.org/nuscenes) | 官方文档 | 数据格式、API使用 |
| [nuScenes Devkit](https://github.com/nutonomy/nuscenes-devkit) | 代码 | Python SDK，必装 |
| [nuscenes-devkit 教程](https://github.com/nutonomy/nuscenes-devkit/blob/master/python-sdk/tutorial.md) | 教程 | 手把手数据解析 |

---

## 核心概念详解

### 1. Transformer (理解程度：能讲清楚即可)

**核心公式 — Scaled Dot-Product Attention**：
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

**Multi-Head Attention**：
$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h)W^O$$
其中 $\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$

**在BEV中的应用**：
- BEVFormer 用 Deformable Attention 替代全局Attention（减少计算量）
- BEV Query 是可学习的BEV网格嵌入，通过 Cross-Attention 查询多相机图像特征

### 2. BEV 坐标系

**BEV = Bird's Eye View (鸟瞰图)**

**定义**：
- 以自车为中心，通常范围 [-50m, 50m] × [-50m, 50m]
- X向前，Y向左 (nuScenes标准)
- 分辨率通常 0.5m~1.0m/pixel

**BEV特征张量**：`[B, C, H, W]`
- B: batch
- C: 通道数 (128~256)
- H, W: BEV网格尺寸 (如 200×200 for 0.5m resolution)

### 3. Image-to-BEV 变换

#### 方法一：LSS (Lift-Splat-Shoot)

```
Lift: 对每个图像像素预测深度分布 (D bins)
      Image Feature [C,H,W] → Depth Distribution [D,H,W]
      → 将2D特征"提升"到3D: Feature*Depth → Pseudo Point Cloud

Splat: 将3D伪点云根据内外参"溅射"到BEV网格
       pillar pooling: 同一BEV cell内的所有点求和/平均
```

**关键公式**：
$$BEV[x,y] = \sum_{pixels\ that\ project\ to\ (x,y)} feature[pixel] \cdot depth\_prob[pixel, d]$$

#### 方法二：BEVFormer (Cross-Attention)

```
1. 定义可学习的 BEV Query Q_bev [H_bev, W_bev, D]
2. 对每个BEV query, 用内外参找到它在各相机图像中的投影位置
3. 在投影位置周围做 Deformable Attention:
   BEV_Query ← CrossAttn(BEV_Query, Image_Features, reference_points)
```

### 4. 时序融合

**BEVDet4D 的时序融合**：

```
Frame t-1: BEV[t-1] 已生成
Frame t:   对齐 + 融合

1. 根据 Ego-motion 将 BEV[t-1] warp 到当前坐标系:
   BEV[t-1]_aligned = warp(BEV[t-1], T_{t←t-1})

2. 与当前帧融合:
   BEV[t]_fused = concat(BEV[t], BEV[t-1]_aligned)
   或用 Temporal Self-Attention
```

### 5. Ego-motion 对齐

**核心操作**：

假设历史 BEV 特征在网格 $(x_{prev}, y_{prev})$ 处，需要变换到当前坐标系：

1. 相对变换：$\Delta T = T_{curr}^{-1} \cdot T_{prev}$
   - 分解为 $(\Delta x, \Delta y, \Delta \theta)$
2. 网格变换：
   $$\begin{bmatrix} x_{curr} \\ y_{curr} \end{bmatrix} = \begin{bmatrix} \cos\Delta\theta & -\sin\Delta\theta \\ \sin\Delta\theta & \cos\Delta\theta \end{bmatrix} \begin{bmatrix} x_{prev} \\ y_{prev} \end{bmatrix} + \begin{bmatrix} \Delta x \\ \Delta y \end{bmatrix}$$
3. 用 `grid_sample` 重采样得到对齐后的特征

**边界处理**：变换后超出BEV范围的位置用mask标记为无效。

### 6. 深度估计与几何先验

**BEVDet4D中的深度**：
- Depth Net 预测每个像素在 D 个深度 bin 上的概率分布
- 监督信号：LiDAR点云投影到图像，one-hot监督

**为什么需要深度？**
- 单张图像无法确定物体的3D位置（尺度歧义）
- 深度预测告诉模型"这个像素对应的物体在多远"
- 深度越准 → BEV投影越准 → 检测越准

### 7. BEV 下游任务

| 任务 | 输出 | 典型方法 |
|------|------|----------|
| 3D 目标检测 | 3D框 + 类别 + 速度 | CenterPoint |
| BEV 语义分割 | 逐grid类别 | BEVDet |
| Occupancy预测 | 体素占用+语义 | SurroundOcc |
| 在线地图 | 车道线、路沿 | MapTR |
| 轨迹预测 | 未来轨迹 | UniAD |

### 8. 与SLAM的连接点

| SLAM能力 | BEV前端中的应用 |
|----------|----------------|
| 相机位姿 | 多帧BEV对齐的Ego-motion |
| 坐标变换 | 相机→车体→BEV→世界的坐标链 |
| 时间同步 | 多帧数据的时间对齐 |
| 标定精度 | 内外参误差→BEV投影误差 |
| 深度/几何 | 稀疏深度作为BEV投影先验 |
| 动态检测 | 标记动态区域，指导时序融合 |

---

## 动手实践清单

- [ ] 安装 nuScenes devkit，加载一个 scene 的数据
- [ ] 将 LiDAR 点投影到 6 路相机图像，验证内外参正确性
- [ ] 实现基本的相机→车体→BEV坐标变换
- [ ] 跑通 BEVDet4D 基线 (单帧 + 多帧)
- [ ] 可视化：深度分布图、BEV特征热力图、3D检测结果
- [ ] 实现历史 BEV 的 Ego-motion 对齐
- [ ] 对比单帧/多帧/有无对齐的 BEV 检测精度
- [ ] 分析位姿噪声对 BEV 特征的影响

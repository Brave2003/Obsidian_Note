# 项目实战：面向端到端自动驾驶的多相机时序BEV感知前端

## 项目总览

```
┌─────────────────────────────────────────────────┐
│              项目输入与输出                        │
├─────────────────────────────────────────────────┤
│ 输入: nuScenes数据集 (6路图像+内外参+Ego Pose)      │
│ 输出: 时序融合的标准化BEV特征张量                    │
│ 基线: BEVDet4D                                    │
│ 增强: VIO Ego-motion 接入                         │
│ 环境: PyTorch + CUDA + nuScenes-devkit            │
└─────────────────────────────────────────────────┘
```

## 环境搭建

### 硬件需求
- GPU: 至少 11GB 显存 (RTX 2080 Ti / RTX 3080 / RTX 4070)
- RAM: 32GB+
- 存储: nuScenes 全量约 350GB，mini版约 4GB

### 软件安装

```bash
# 1. 创建conda环境
conda create -n bev python=3.8
conda activate bev

# 2. PyTorch (CUDA 11.x)
pip install torch==1.13.1 torchvision==0.14.1

# 3. nuScenes devkit
pip install nuscenes-devkit

# 4. MMDetection3D (如果使用mmdet3d版本的BEVDet)
pip install mmcv-full==1.6.0 -f https://download.openmmlab.com/mmcv/dist/cu113/torch1.13.0/index.html
pip install mmdet==2.25.1
pip install mmsegmentation==0.25.0
git clone https://github.com/open-mmlab/mmdetection3d.git
cd mmdetection3d && pip install -e .

# 5. BEVDet4D (直接克隆开源实现)
git clone https://github.com/HuangJunJie2017/BEVDet
cd BEVDet
pip install -e .
```

---

## 阶段一：数据与几何管线

### Step 1: 理解 nuScenes 数据结构

```python
from nuscenes.nuscenes import NuScenes
nusc = NuScenes(version='v1.0-mini', dataroot='./data/nuscenes', verbose=True)

# 数据结构
scene = nusc.scene[0]          # 一段连续驾驶片段 (约20s)
sample = nusc.sample[0]        # 一个时间点
sample_data = nusc.get('sample_data', sample['data']['CAM_FRONT'])

# 获取标定
cs_record = nusc.get('calibrated_sensor', sample_data['calibrated_sensor_token'])
# cs_record['translation']  # 相机在车体系中的位置
# cs_record['rotation']     # 相机在车体系中的旋转

# 获取Ego Pose
ep_record = nusc.get('ego_pose', sample_data['ego_pose_token'])
# ep_record['translation']  # 车体在全局系中的位置
# ep_record['rotation']     # 车体在全局系中的旋转

# 6路相机
CAM_NAMES = ['CAM_FRONT', 'CAM_FRONT_LEFT', 'CAM_FRONT_RIGHT',
             'CAM_BACK', 'CAM_BACK_LEFT', 'CAM_BACK_RIGHT']
```

### Step 2: 坐标变换实现

```python
import numpy as np
from pyquaternion import Quaternion

def get_camera_intrinsics(cs_record):
    """获取相机内参矩阵 3x3"""
    # nuscenes用针孔模型，无畸变
    fx = fy = ...  # 从cs_record计算
    return np.array([[fx, 0, cx],
                     [0, fy, cy],
                     [0,  0,  1]])

def get_sensor2ego(cs_record):
    """传感器坐标系 → 车体坐标系"""
    translation = np.array(cs_record['translation'])
    rotation = Quaternion(cs_record['rotation'])
    T = np.eye(4)
    T[:3, :3] = rotation.rotation_matrix
    T[:3, 3] = translation
    return T  # sensor → ego

def get_ego2global(ep_record):
    """车体坐标系 → 全局坐标系"""
    translation = np.array(ep_record['translation'])
    rotation = Quaternion(ep_record['rotation'])
    T = np.eye(4)
    T[:3, :3] = rotation.rotation_matrix
    T[:3, 3] = translation
    return T  # ego → global

# 完整的坐标链
# camera → ego → global → ego' → camera' →
# BEV (以当前时刻自车为中心的鸟瞰图)
```

### Step 3: LiDAR点投影到图像（验证标定）

```python
def project_lidar_to_camera(points_lidar, T_cam_lidar, K):
    """
    points_lidar: [N, 3] LiDAR坐标系下的点
    T_cam_lidar: 4x4, LiDAR → Camera 变换
    K: 3x3 内参
    返回: [N, 2] 像素坐标 (有效点)
    """
    # 变换到相机系
    pts_homo = np.hstack([points_lidar, np.ones((N, 1))])
    pts_cam = (T_cam_lidar @ pts_homo.T)[:3]  # [3, N]

    # 投影
    pts_img = K @ pts_cam  # [3, N]
    pts_img = pts_img[:2] / pts_img[2]  # 归一化

    # 过滤：在前面、在图像内
    valid = (pts_cam[2] > 0) & (pts_img[0] > 0) & (pts_img[0] < W) & ...
    return pts_img[:, valid]
```

### 验收检查清单

- [ ] 能读取 nuScenes mini 的一个完整 scene
- [ ] 能将 LiDAR 点正确投影到 6 路相机图像
- [ ] 能画出某时刻 6 路相机图像的 BEV 视锥范围
- [ ] 能解释每个变换矩阵的方向 ($T_{cam}^{ego}$, $T_{ego}^{global}$)

---

## 阶段二：跑通 BEVDet4D 基线

### 最小可跑配置

```python
# configs/bevdet4d-nuscenes.py 关键参数
img_backbone = dict(type='ResNet', depth=50)           # 图像backbone
img_neck = dict(type='FPN', ...)                       # 多尺度颈部
depth_net = dict(...)                                   # 深度分布预测
view_transform = dict(type='LSSViewTransformer')        # LSS投影
bev_encoder = dict(type='ResNet', depth=18, in_channels=...)  # BEV编码
```

### 运行命令

```bash
# 训练 (单帧)
python tools/train.py configs/bevdet-nuscenes.py

# 训练 (多帧, BEVDet4D)
python tools/train.py configs/bevdet4d-nuscenes.py

# 推理 + 可视化
python tools/test.py configs/bevdet4d-nuscenes.py ckpt.pth --show
```

### 需要输出的可视化

1. **6路相机输入**：原始图像
2. **深度分布**：每像素的深度概率热力图
3. **BEV特征**：某通道的BEV特征图
4. **3D检测结果**：BEV视角下的3D框

### 记录指标

| 指标 | 单帧 | 多帧(无对齐) | 多帧(有对齐) |
|------|------|-------------|-------------|
| NDS | | | |
| mAP | | | |
| 推理时间 (ms) | | | |
| 显存 (GB) | | | |

---

## 阶段三：历史BEV位姿对齐 (核心)

### 实现步骤

```python
def align_history_bev(bev_prev, ego_pose_prev, ego_pose_curr, bev_range, bev_resolution):
    """
    bev_prev: [C, H, W] 历史BEV特征
    ego_pose_prev: 4x4, 历史时刻的ego pose (ego→global)
    ego_pose_curr: 4x4, 当前时刻的ego pose (ego→global)
    bev_range: (min_x, max_x, min_y, max_y)
    bev_resolution: 每个grid的米数
    """
    # 1. 计算相对变换
    T_prev2curr = np.linalg.inv(ego_pose_curr) @ ego_pose_prev

    # 2. 提取2D变换 (BEV在水平面上)
    dx = T_prev2curr[0, 3]   # x方向平移
    dy = T_prev2curr[1, 3]   # y方向平移
    dtheta = np.arctan2(T_prev2curr[1, 0], T_prev2curr[0, 0])  # yaw旋转

    # 3. 生成采样网格
    # 对当前BEV的每个grid位置，找它在历史BEV中对应的位置
    H, W = bev_prev.shape[1:]
    grid_y, grid_x = torch.meshgrid(torch.arange(H), torch.arange(W))

    # 当前坐标 → 世界偏移
    x_curr = (grid_x * bev_resolution + bev_range[0])
    y_curr = (grid_y * bev_resolution + bev_range[2])

    # 世界偏移 → 历史坐标 (逆变换)
    x_prev = (x_curr - dx) * torch.cos(dtheta) + (y_curr - dy) * torch.sin(dtheta)
    y_prev = -(x_curr - dx) * torch.sin(dtheta) + (y_curr - dy) * torch.cos(dtheta)

    # 转到grid坐标
    grid_x_prev = (x_prev - bev_range[0]) / bev_resolution
    grid_y_prev = (y_prev - bev_range[2]) / bev_resolution

    # 归一化到[-1, 1]
    grid = torch.stack([grid_x_prev*2/H-1, grid_y_prev*2/W-1], dim=-1)

    # 4. 采样
    bev_aligned = F.grid_sample(bev_prev.unsqueeze(0), grid.unsqueeze(0),
                                 mode='bilinear', padding_mode='zeros', align_corners=True)

    # 5. 生成有效掩码
    valid_mask = (grid_x_prev >= 0) & (grid_x_prev < W) & \
                 (grid_y_prev >= 0) & (grid_y_prev < H)

    return bev_aligned.squeeze(0), valid_mask
```

### 融合方式

```python
# 简单方案：加权平均
bev_fused = bev_curr * w_curr + bev_aligned * w_prev

# 进阶方案：可学习的时序融合
bev_fused = temporal_fusion_module(
    torch.cat([bev_curr, bev_aligned], dim=0)
)
```

### 对比实验矩阵

```
                    | 单帧 | 2帧 | 4帧 | 8帧
--------------------|------|-----|-----|-----
无位姿对齐           |  ✓   |  ✓  |  ✓  |
真值Ego Pose对齐     |  ✓   |  ✓  |  ✓  |
真值+噪声(0.01m,0.01°) |    |  ✓  |     |
真值+噪声(0.05m,0.05°) |    |  ✓  |     |
真值+噪声(0.10m,0.10°) |    |  ✓  |     |
VIO估计位姿对齐       |      |  ✓  |     |
```

---

## 阶段四：几何增强 (推荐方案B：VIO Ego-motion接入)

### 为什么推荐这个方案？

1. **直接连接你的SLAM背景** — 这是你最有优势的方向
2. **回答核心问题** — 定位误差对BEV影响多大？工业界的实际关切
3. **差异化** — 纯DL背景的人做不了这个实验

### 实现步骤

```python
# Step 1: 在nuScenes数据上用VINS/OpenVINS跑VIO
# 输出：VIO估计的每帧ego pose

# Step 2: 与真值比较，量化VIO精度
def compute_relative_pose_error(pose_est, pose_gt):
    """计算相邻帧之间的相对位姿误差"""
    delta_est = np.linalg.inv(pose_est[t]) @ pose_est[t+1]
    delta_gt = np.linalg.inv(pose_gt[t]) @ pose_gt[t+1]
    delta_err = np.linalg.inv(delta_gt) @ delta_est
    trans_err = np.linalg.norm(delta_err[:3, 3])
    rot_err = np.arccos((np.trace(delta_err[:3, :3]) - 1) / 2)
    return trans_err, rot_err

# Step 3: 分别用真值和VIO位姿做BEV对齐
# Step 4: 比较下游任务精度
```

### 关键分析维度

- **位姿误差 vs BEV特征错位**：量化关系
- **平移 vs 旋转**：哪个更敏感？
- **车速影响**：快车vs慢车，位姿误差影响不同
- **转弯 vs 直行**：旋转误差在转弯时更致命

---

## 阶段五：统一输出接口

```python
class BEVFrontend(nn.Module):
    def __init__(self, config):
        self.backbone = build_backbone(config)
        self.neck = build_neck(config)
        self.depth_net = DepthNet(config)
        self.view_transform = LSSViewTransform(config)
        self.bev_encoder = BEVEncoder(config)
        self.temporal_align = TemporalAlignment(config)

    def forward(self, images, intrinsics, extrinsics, ego_poses, timestamps):
        """
        Args:
            images: [B, N_cam, 3, H, W] 多相机图像
            intrinsics: [B, N_cam, 3, 3] 相机内参
            extrinsics: [B, N_cam, 4, 4] 相机→车体外参
            ego_poses: [B, T, 4, 4] 历史ego pose (含当前帧)
            timestamps: [B, T] 时间戳
        Returns:
            dict:
                - bev_features: [B, C, H_bev, W_bev]
                - bev_range: (min_x, max_x, min_y, max_y)
                - bev_resolution: float
                - ego_pose_current: [B, 4, 4]
                - valid_mask: [B, H_bev, W_bev]
                - confidence: [B, H_bev, W_bev] (深度/几何置信度)
                - quality_flag: dict (数据质量标记)
        """
        pass
```

---

## 最终产出清单

- [ ] GitHub 仓库，含完整 README
- [ ] 环境配置脚本 (environment.yml / requirements.txt)
- [ ] nuScenes 数据管线代码
- [ ] 坐标投影和 BEV 可视化工具
- [ ] 可运行的 BEVDet4D 基线 (单帧 + 多帧)
- [ ] 历史 BEV 位姿对齐模块
- [ ] VIO 位姿接入实验
- [ ] 完整消融实验表格
- [ ] 失败场景分析文档
- [ ] 系统框架图 (draw.io 或 Excalidraw)
- [ ] 2-3 分钟演示视频 (屏幕录制 + 解说)
- [ ] 一页技术总结 (PDF)

# Nonlinear 非线性优化模块

GTSAM中的`nonlinear`模块包含了一套全面的工具，用于使用因子图(factor graph)进行非线性优化。以下是按类别组织的关键组件概述：

## 核心类

- [NonlinearFactorGraph](doc/NonlinearFactorGraph.ipynb): 将优化问题表示为因子图。
- [NonlinearFactor](doc/NonlinearFactor.ipynb): 所有非线性因子的基类。
- [NoiseModelFactor](doc/NonlinearFactor.ipynb): 带有噪声模型(noise model)的因子基类。
- [Values](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/Values.h): 用于优化的变量赋值容器。

## 批量优化器(Batch Optimizers)

- [NonlinearOptimizer](doc/NonlinearOptimizer.ipynb): 所有批量优化器的基类。
    - [NonlinearOptimizerParams](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearOptimizerParams.h): 所有优化器的基类参数。

- [GaussNewtonOptimizer](doc/GaussNewtonOptimizer.ipynb): 实现高斯-牛顿(Gauss-Newton)优化。
    - [GaussNewtonParams](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GaussNewtonParams.h): 高斯-牛顿优化的参数。

- [LevenbergMarquardtOptimizer](doc/LevenbergMarquardtOptimizer.ipynb): 实现列文伯格-马夸尔特(Levenberg-Marquardt)优化。
    - [LevenbergMarquardtParams](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/LevenbergMarquardtParams.h): 列文伯格-马夸尔特优化的参数。

- [DoglegOptimizer](doc/DoglegOptimizer.ipynb): 实现Powell的Dogleg优化。
    - [DoglegParams](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/DoglegParams.h): Dogleg优化的参数。

- [GncOptimizer](doc/GncOptimizer.ipynb): 使用渐进非凸(Graduated Non-Convexity)方法实现鲁棒优化。
    - 对于光束法平差(bundle adjustment)问题上的GPU加速GNC，请参见[GNC with the CUDA SFM optimizer](../slam/doc/CudaSfmGncOptimizer.ipynb)。
    - [GncParams](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GncParams.h): 渐进非凸优化的参数。

## 增量优化器(Incremental Optimizers)

- [ISAM2](doc/ISAM2.ipynb): 增量平滑与建图2(Incremental Smoothing and Mapping 2)，具有流式重线性化(fluid relinearization)功能。
    - [ISAM2Params](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/ISAM2Params.h): 控制ISAM2算法的参数。
    - [ISAM2Result](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/ISAM2Result.h): ISAM2更新操作的结果。
- [NonlinearISAM](doc/NonlinearISAM.ipynb): 原始iSAM实现（大部分已被ISAM2取代）。

## 特殊因子

- [PriorFactor](doc/PriorFactor.ipynb): 对变量施加先验约束。
- [NonlinearEquality](doc/NonlinearEquality.ipynb): 强制变量之间的等式约束。
- [LinearContainerFactor](doc/LinearContainerFactor.ipynb): 将线性因子包装以包含在非线性因子图中。
- [WhiteNoiseFactor](doc/WhiteNoiseFactor.ipynb): 用于估计零均值高斯白噪声参数的二值因子。

## 滤波和平滑(Filtering and Smoothing)

- [ExtendedKalmanFilter](doc/ExtendedKalmanFilter.ipynb): 非线性卡尔曼滤波器实现。
- [FixedLagSmoother](doc/FixedLagSmoother.ipynb): 固定滞后平滑器的基类。
    - [BatchFixedLagSmoother](doc/BatchFixedLagSmoother.ipynb): 使用批量优化的固定滞后平滑器实现。
    - [IncrementalFixedLagSmoother](doc/IncrementalFixedLagSmoother.ipynb): 使用iSAM2的固定滞后平滑器实现。

## 分析和可视化

- [Marginals](doc/Marginals.ipynb): 从优化结果计算边缘协方差和联合边缘分布。
- [`Marginals.h`](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/Marginals.h): `Marginals`和`JointMarginal`的C++ API声明。
- [GraphvizFormatting](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GraphvizFormatting.h): 为因子图可视化提供自定义功能。

## BatchFixedLagSmoother
# BatchFixedLagSmoother

## 概述

`IncrementalFixedLagSmoother`是一个使用[LevenbergMarquardtOptimizer](LevenbergMarquardtOptimizer.ipynb)进行批量优化的[FixedLagSmoother](FixedLagSmoother.ipynb)。

此固定滞后平滑器将在每次迭代时**批量优化**，但从上次估计值热启动(warm-started)。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/BatchFixedLagSmoother.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## API

### 构造函数

您使用以下参数构造一个`BatchFixedLagSmoother`对象：

- **smootherLag**: 平滑器滞后长度。任何比此更旧的变量将被边缘化出去。*(默认: 0.0)*
- **parameters**: 列文伯格-马夸尔特优化参数。*(默认: `LevenbergMarquardtParams()`)*
- **enforceConsistency**: 一个标志，指示优化器是否应通过保持平滑窗口边缘处所有涉及线性化/边缘化因子的变量的线性化点来强制概率一致性。*(默认: `true`)*

### 平滑和优化

- **update**: 此方法是`BatchFixedLagSmoother`的核心。它处理新的因子和变量，更新当前的状态估计。update方法还管理落在固定滞后窗口外的变量的边缘化。

### 计算考虑

每次对`update`的调用都会触发批量LM优化：使用参数控制收敛阈值，以将计算限制在适合您应用的范围。

## 内部实现

- **marginalize**: 此函数处理不再在固定滞后窗口内的变量的边缘化。边缘化是维持因子图大小的关键步骤，确保只保留相关变量进行优化。

## ConcentratedGaussian
# ConcentratedGaussian

GTSAM中用于表示流形值类型上（可能平移的）高斯密度的统一概率原语。本笔记服务于三类读者：

1. 只需要类似概率对象的通用GTSAM用户。
2. 接收`ConcentratedGaussian`输出的EKF用户。
3. 利用左扩展集中高斯(L-ECG, Left Extended Concentrated Gaussian)的传输、重置和融合的高级用户。

相关笔记：[PriorFactor](PriorFactor.ipynb), [ExtendedPriorFactor](ExtendedPriorFactor.ipynb)。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ConcentratedGaussian.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
# Install GTSAM and Plotly from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass # Not in Colab
```

```python
import math
from typing import TypeAlias

import numpy as np
import plotly.graph_objects as go
from plotly.subplots import make_subplots

import gtsam
from gtsam import Point2, Point3, Pose2, Rot2

# Type aliases for clarity when passing to GTSAM
V: TypeAlias = np.ndarray  # Vector (1D) passed into GTSAM
M: TypeAlias = np.ndarray  # Matrix (2D) passed into GTSAM
```

## 1. General Usage 常规用法

`ConcentratedGaussian<T>`的行为类似于流形变量`T`上的连续概率密度。对于高斯噪声模型，它计算到正确归一化的概率（受浮点限制）：

- 使用原点`o`、协方差`Sigma`（或噪声模型）和可选的切空间均值`m`构造。
- 内部误差项：`e(x) = -Local(x, o) - m`（如果提供了均值）。
- 概率：`P(x) = exp( logProbability(x) )`。
- `evaluate(x)`是`P(x)`的便捷同义词。

我们将从一个使用`Point2`（欧几里得）的简单2D示例开始。

```python
# Zero-mean Point2 density
origin: V = Point2(0.0, 0.0)
cov: M = np.array([[0.4, 0.1],
                   [0.1, 0.2]], dtype=float)
density = gtsam.ConcentratedGaussianPoint2(1, origin, cov)

xs: V = np.linspace(-2,2,160)
ys: V = np.linspace(-2,2,160)
X, Y = np.meshgrid(xs, ys)
Z: M = np.zeros_like(X)
for i in range(X.shape[0]):
    for j in range(X.shape[1]):
        Z[i,j] = density.evaluate(Point2(X[i,j], Y[i,j]))

# NOTE: In the Python wrapper Point2 is a numpy ndarray alias, so we use p[0], p[1].
fig = go.Figure()
fig.add_trace(go.Contour(x=xs, y=ys, z=Z, colorscale='Viridis', contours=dict(showlabels=True)))
fig.add_trace(go.Scatter(x=[origin[0]], y=[origin[1]], mode='markers', marker=dict(color='red', size=10), name='origin'))
fig.update_layout(title='Point2 ConcentratedGaussian (zero-mean)', xaxis_title='x', yaxis_title='y')
fig.show()
```

### 添加切空间均值

非零均值将模式(mode)移动到`origin.retract(mean)`（Point2：就是加法）。

```python
# Non-zero tangent mean shifts the mode to origin.retract(mean)
mean: V = np.array([0.8, -0.4], dtype=float)
density_shifted = gtsam.ConcentratedGaussianPoint2(2, origin, mean, cov)
mode_point: V = origin + mean  # numpy-based Point2 + mean
mode_prob = density_shifted.evaluate(Point2(mode_point[0], mode_point[1]))
print('Mode probability (approx peak):', mode_prob)

Zs: M = np.zeros_like(Z)
for i in range(X.shape[0]):
    for j in range(X.shape[1]):
        Zs[i,j] = density_shifted.evaluate(Point2(X[i,j], Y[i,j]))

fig2 = go.Figure()
fig2.add_trace(go.Contour(x=xs, y=ys, z=Zs, colorscale='Inferno', contours=dict(showlabels=True)))
fig2.add_trace(go.Scatter(x=[origin[0]], y=[origin[1]], mode='markers', marker=dict(color='red', size=10), name='origin'))
fig2.add_trace(go.Scatter(x=[mode_point[0]], y=[mode_point[1]], mode='markers', marker=dict(color='green', size=10, symbol='x'), name='mode'))
fig2.update_layout(title='Shifted ConcentratedGaussian (Point2)', xaxis_title='x', yaxis_title='y')
fig2.show()
```

### 带有样本的Pose2示例（香蕉形分布）

让我们模拟一个车辆走N步，但在角度上有很大的不确定性。为获得明显的"香蕉"分布，我们模拟许多短距离前进运动增量并带有航向噪声。每个样本是小步的随机游走：

1. 从单位元位姿开始。
2. 对K步中的每一步，采样一个小的前向平移（带有少量横向漂移）和航向扰动。
3. 复合这些增量位姿。
4. 记录最终的(x,y,theta)。

累积的航向噪声弯曲轨迹，产生在(x,y)上的弯曲（香蕉形）边缘分布。

```python
def plot_pose2_samples(title: str, poses: list, trajectory: list = None) -> go.Figure:
  xs, ys, thetas = zip(*[(p.x(), p.y(), np.rad2deg(p.theta())) for p in poses])
  fig = go.Figure()
  fig.add_trace(go.Scattergl(
    x=xs, y=ys, mode='markers',
    marker=dict(size=3, opacity=0.35, color=thetas, colorscale='Turbo', colorbar=dict(title='theta')),
    name='end poses'))
  if trajectory is not None:
    # Draw tiny arrows for each pose in trajectory
    for p in trajectory:
      x = p.x()
      y = p.y()
      th = p.theta()
      dx = 0.03 * np.cos(th)
      dy = 0.03 * np.sin(th)
      fig.add_trace(go.Scatter(
        x=[x, x + dx], y=[y, y + dy],
        mode='lines',
        line=dict(color='black', width=1),
        showlegend=False
      ))
  fig.update_layout(
    title=title,
    xaxis_title='x', yaxis_title='y', yaxis_scaleanchor='x'
  )
  fig.update_layout(margin=dict(l=0, r=0, t=40, b=0))
  return fig
```

```python
# Incremental random-walk generation of a banana-shaped distribution
rng = np.random.default_rng(1)
N: int = 5000   # number of trajectories
K: int = 20     # number of small increments per trajectory

# Constant twist: mean increment is a fixed 3-vector (vx, vy, omega)
mean_twist :V = np.array([0.12, 0.0, np.deg2rad(2)], dtype=float)  # forward, lateral, angular velocity (2 deg/step)
cov_twist :M = np.diag([0.01**2, 0.01**2, np.deg2rad(5)**2])      # diagonal covariance

poses = []
for n in range(N):
    pose = Pose2(0, 0, 0)
    for _ in range(K):
        delta :V = rng.multivariate_normal(mean_twist, cov_twist)
        pose = pose.retract(delta)
    poses.append(pose)

pose = Pose2(0, 0, 0)
trajectory = [pose]
for _ in range(K):
    pose = pose.retract(mean_twist)
    trajectory.append(pose)
plot_pose2_samples("Pose2 constant twist + noise, banana-shaped distribution", poses, trajectory)
```

正如RSS 2008论文"The Banana Distribution is Gaussian"中所述，我们现在可以对*指数坐标*拟合一个高斯分布：

```python
# Stack xs, ys, thetas into Pose2s, Logmap to tangent vectors, fit mean/covariance
tangent_vecs = np.array([Pose2.Logmap(p) for p in poses])
mean_tangent : V = np.mean(tangent_vecs, axis=0)
cov_tangent : M = np.cov(tangent_vecs, rowvar=False)

# Create a ConcentratedGaussianPose2 using mean_tangent and cov_tangent at origin_pose
density_pose2 = gtsam.ConcentratedGaussianPose2(20, Pose2(), mean_tangent, cov_tangent)
print(density_pose2)
```

```python
# Sample from the tangent Gaussian, push through ExpMap, and plot (x, y, theta)
maybe_mP = density_pose2.gaussian()
assert maybe_mP is not None, "Expected gaussian() to return a value"
m, P = maybe_mP  # P=covariance, m=mean
samples_tangent = rng.multivariate_normal(m, P, size=N)

poses_push = []
for v in samples_tangent:
  p = Pose2.Expmap(v)
  poses_push.append(p)

plot_pose2_samples("Samples from fitted ConcentratedGaussianPose2", poses_push, trajectory)
```

香蕉形状关键取决于具有远离原点均值的高斯分布：Pose2中的测地线在x-y平面中追踪出圆圈，因此`ConcentratedGaussian`提供的*偏移*高斯能够很好地近似香蕉分布。

我们甚至可以解析地预测高斯分布：

```python
# Compute the analytic mean and covariance in tangent space
mean_analytic = Pose2.Logmap(trajectory[-1])
cov_analytic = K * cov_twist  # This is an approximation; for true analytic, propagate with Jacobians
analytic = gtsam.ConcentratedGaussianPose2(20, Pose2(), mean_analytic, cov_analytic)

# Sample from the analytically obtained Gaussian
maybe_mP = analytic.gaussian()
assert maybe_mP is not None, "Expected gaussian() to return a value"
m, P = maybe_mP  # P=covariance, m=mean
samples_tangent = rng.multivariate_normal(m, P, size=N)

poses_push = []
for v in samples_tangent:
  p = Pose2.Expmap(v)
  poses_push.append(p)

plot_pose2_samples("Samples from an analytically obtained ConcentratedGaussianPose2", poses_push, trajectory)
```

## 2. EKF视角

扩展卡尔曼滤波器(EKF)的后验可以表示为`ConcentratedGaussian`：

- 原点(Origin): 线性化/参考状态。
- 切空间均值(Tangent mean): 从原点到模式的偏移。
- 协方差(Covariance): 该切空间中的不确定性。

在将此后验用作下一次预测的先验之前，调用`reset()`将原点移动到模式并将均值清零（保持协方差已传输）。

```python
# Simulated EKF posterior with non-zero tangent mean
origin_post = Pose2(1.0, 2.0, 0.2)
cov_post: M = np.diag([0.05, 0.04, 0.02]).astype(float)
mean_post: V = np.array([0.3, -0.1, 0.15], dtype=float)
posterior = gtsam.ConcentratedGaussianPose2(5, origin_post, mean_post, cov_post)
mode_pose = posterior.retractMean()
print('Origin:', origin_post)
print('Mode  :', mode_pose)
print('Mode probability:', posterior.evaluate(mode_pose))

reset_density = posterior.reset()
print('After reset -> origin == mode?', reset_density.origin() == mode_pose)
print('Reset mean (should be empty/None):', reset_density.mean())

fig_ekf = go.Figure()
fig_ekf.add_trace(go.Scatter(x=[origin_post.x()], y=[origin_post.y()], mode='markers',
                             marker=dict(color='red', size=10), name='origin'))
fig_ekf.add_trace(go.Scatter(x=[mode_pose.x()], y=[mode_pose.y()], mode='markers',
                             marker=dict(color='green', size=10, symbol='x'), name='mode'))
fig_ekf.update_layout(title='EKF Posterior: Origin vs Mode', xaxis_title='x', yaxis_title='y')
fig_ekf.show()
```

## 3. Advanced Operations: L-ECG Mechanics 高级操作：L-ECG机制

`ConcentratedGaussian`实现了左扩展集中高斯(L-ECG, Left Extended Concentrated Gaussian)。高级能力：

1. `reset()` -- 将原点移动到模式，将切空间均值清零，传输协方差。
2. `transportTo(x_hat)` -- 在新的坐标卡（新原点）中表示密度，引发均值和传输后的协方差。
3. `operator*` -- 两个密度（相同键）在公共坐标卡中的近似融合，随后重置。

下面：Pose2的传输和融合示例。

```python
# Transport & Fusion with separated, differently oriented densities
originA = Pose2(-2.0, 0.0, math.pi/4)
covA: M = np.diag([0.25, 0.12, 0.10]).astype(float)
meanA: V = np.array([0.4, 0.0, 0.0], dtype=float)

originB = Pose2(2.0, 0.0, -math.pi/3)
covB: M = np.diag([0.25, 0.12, 0.10]).astype(float)
meanB: V = np.array([0.4, 0.0, 0.0], dtype=float)

dA = gtsam.ConcentratedGaussianPose2(10, originA, meanA, covA)
dB = gtsam.ConcentratedGaussianPose2(10, originB, meanB, covB)

fused = dA * dB
print('Fused origin:', fused.origin())
print('Fused mean (should be None):', fused.mean())
```

```python
# Plot the things
fig_fusion = go.Figure()
def heading_arrow(pose, length: float = 0.7, color: str = 'black', name: str = 'heading') -> None:
    x = pose.x(); y = pose.y(); th = pose.theta()
    x2 = x + length*math.cos(th)
    y2 = y + length*math.sin(th)
    fig_fusion.add_trace(go.Scatter(x=[x, x2], y=[y, y2], mode='lines', line=dict(color=color, width=3), name=name, showlegend=False))

fig_fusion.add_trace(go.Scatter(x=[originA.x()], y=[originA.y()], mode='markers', marker=dict(color='blue', size=10), name='Origin A'))
fig_fusion.add_trace(go.Scatter(x=[originB.x()], y=[originB.y()], mode='markers', marker=dict(color='red', size=10), name='Origin B'))
fig_fusion.add_trace(go.Scatter(x=[fused.origin().x()], y=[fused.origin().y()], mode='markers', marker=dict(color='green', size=12, symbol='x'), name='Fused'))

heading_arrow(originA, color='blue')
heading_arrow(originB, color='red')
heading_arrow(fused.origin(), color='green')

fig_fusion.update_layout(title='Fusion of Two Opposed Pose2 Densities (Result ~ centered, heading ~ 0)',
                         xaxis_title='x', yaxis_title='y', yaxis_scaleanchor='x')
fig_fusion.show()
```

### Caveats 注意事项

- 融合和传输是一阶的；对于大分离度，考虑迭代细化。
- 大的切空间均值可能意味着需要重新中心化(`reset`)以保持高斯保真度。
- 当前示例在采样时在外部跟踪协方差；在Python中暴露底层高斯模型将使高级可视化更容易。

## Summary 总结

`ConcentratedGaussian` = 流形感知的高斯密度：基本概率求值（第1节），带有`retractMean()`/`reset()`的EKF后验处理（第2节），以及用于高级工作流程的几何操作（传输、融合）（第3节）。选择解决您任务的最简单接口。

## CustomFactor
# CustomFactor

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/CustomFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

## 概述

`CustomFactor`类允许用户定义自定义误差函数和Jacobian，虽然它可以在C++中使用，但对于在Python封装器中使用尤其有用。

## 自定义误差函数

`CustomFactor`类允许用户定义自定义误差函数。在C++中定义如下：

```cpp
using JacobianVector = std::vector<Matrix>;
using CustomErrorFunction = std::function<Vector(const CustomFactor &, const Values &, const JacobianVector *)>;
```

该函数将接收对因子本身的引用以便可以访问键、`Values`引用和一个可写的Jacobian向量。

## 在Python中使用

要使用基于Python的因子，需要有一个具有以下签名的Python函数：

```python
def error_func(this: gtsam.CustomFactor, v: gtsam.Values, H: list[np.ndarray]) -> np.ndarray:
    ...
```

**说明**：
- `this`是对`CustomFactor`对象的引用。这需要是因为同一个`error_func`可以复用于多个因子。`v`是对当前值集合的引用，`H`是对所需Jacobian列表的*引用*列表（参见相应的C++文档）。
- 返回的误差必须是一个1D `numpy`数组。
- 如果`H`是`None`，意味着当前因子求值不需要Jacobian。例如，因子上的`error`方法不需要Jacobian，所以不计算它们以节省CPU。如果`H`不是`None`，`H`的每个条目可以赋值一个(2D)`numpy`数组，作为对应变量的Jacobian。
- 所有内部的`numpy`矩阵应使用`order="F"`以保持与C++的互操作性。

定义`error_func`后，可以像GTSAM中任何其他因子一样创建`CustomFactor`。总的来说，要使用`CustomFactor`，用户必须：
1. 定义对特定测量或约束建模的自定义误差函数。
2. 实现误差函数的Jacobian矩阵的计算。
3. 定义适当维度的噪声模型。
4. 将`CustomFactor`添加到因子图中，指定：
    - 噪声模型
    - 它依赖的变量的键
    - 误差函数

**注意事项**：
- 对该函数没有太多限制，但注意从C++优化循环中调用Python函数会产生开销。
- 由于`pybind11`需要锁定Python GIL锁来求值每个因子，`CustomFactor`的并行求值是不可能的。
- 您可以通过使用利用测量批处理的Python函数来减轻这两个问题。

在Python中使用的一些更多示例见[test_custom_factor.py](https://github.com/borglab/gtsam/blob/develop/python/gtsam/tests/test_custom_factor.py)、[CustomFactorExample.py](https://github.com/borglab/gtsam/blob/develop/python/gtsam/examples/CustomFactorExample.py)和[CameraResectioning.py](https://github.com/borglab/gtsam/blob/develop/python/gtsam/examples/CameraResectioning.py)。

## 示例
下面是一个模仿`BetweenFactor<Pose2>`的简单示例。

```python
import numpy as np
from gtsam import CustomFactor, noiseModel, Values, Pose2

measurement = Pose2(2, 2, np.pi / 2) # is used to create the error function

def error_func(this: CustomFactor, v: Values, H: list[np.ndarray]=None):
    """
    Error function that mimics a BetweenFactor
    :param this: reference to the current CustomFactor being evaluated
    :param v: Values object
    :param H: list of references to the Jacobian arrays
    :return: the non-linear error
    """
    key0 = this.keys()[0]
    key1 = this.keys()[1]
    gT1, gT2 = v.atPose2(key0), v.atPose2(key1)
    error = measurement.localCoordinates(gT1.between(gT2))

    if H is not None:
        result = gT1.between(gT2)
        H[0] = -result.inverse().AdjointMap()
        H[1] = np.eye(3)
    return error

# we use an isotropic noise model, and keys 66 and 77
noise_model = noiseModel.Isotropic.Sigma(3, 0.1)
custom_factor = CustomFactor(noise_model, [66, 77], error_func)
print(custom_factor)
```

通常，您不会直接调用自定义因子的方法：非线性优化器将在每次非线性迭代中调用`linearize`。但如果您想这样做，以下是方法：

```python
values = Values()
values.insert(66, Pose2(1, 2, np.pi / 2))
values.insert(77, Pose2(-1, 4, np.pi))

print("error = ", custom_factor.error(values))
print("Linearized JacobianFactor:\n", custom_factor.linearize(values))
```

## 注意Jacobian！

单元测试您提供的Jacobian很重要，因为GTSAM使用的约定经常导致混淆。特别地，GTSAM使用*右侧*指数映射更新变量。对于一个n维李群的变量$x\in G$，在$x=a$处的Jacobian $H_a$定义为满足下式的线性映射：
$$
\lim_{\xi\rightarrow0}\frac{\left|f(a)+H_a\xi-f\left(a \, \text{Exp}(\xi)\right)\right|}{\left|\xi\right|}=0,
$$
其中$\xi$是对应于李代数$\mathfrak{g}$中元素的n维向量，$\text{Exp}(\xi)\doteq\exp(\xi^{\wedge})$，$\exp$是从$\mathfrak{g}$回到$G$的指数映射。对于n维流形$M$同样适用，在这种情况下我们使用合适的retraction代替指数映射。更多细节和示例可以在[doc/math.pdf](https://github.com/borglab/gtsam/blob/develop/gtsam/doc/math.pdf)中找到。

要测试您的Jacobian，可以使用方便的`gtsam.utils.numerical_derivative`模块。下面给出示例：

```python
from gtsam.utils.numerical_derivative import numericalDerivative21, numericalDerivative22

# Allocate the Jacobians and call error_func
H = [np.empty((6, 6), order='F'),np.empty((6, 6), order='F')]
error_func(custom_factor, values, H)

# We use error_func directly, so we need to create a binary function constructing the values.
def f (T1, T2):
    v = Values()
    v.insert(66, T1)
    v.insert(77, T2)
    return error_func(custom_factor, v)
numerical0 = numericalDerivative21(f, values.atPose2(66), values.atPose2(77))
numerical1 = numericalDerivative22(f, values.atPose2(66), values.atPose2(77))

# Check the numerical derivatives against the analytical ones
np.testing.assert_allclose(H[0], numerical0, rtol=1e-5, atol=1e-8)
np.testing.assert_allclose(H[1], numerical1, rtol=1e-5, atol=1e-8)
```

## 实现说明

`CustomFactor`是一个`NonlinearFactor`，其回调是`std::function`。得益于`pybind11`的函数支持，此回调可以转换为Python函数调用。

`CustomFactor`的构造函数为：
```cpp
/**
* Constructor
* @param noiseModel shared pointer to noise model
* @param keys keys of the variables
* @param errorFunction the error functional
*/
CustomFactor(const SharedNoiseModel& noiseModel, const KeyVector& keys, const CustomErrorFunction& errorFunction) :
  Base(noiseModel, keys) {
  this->error_function_ = errorFunction;
}
```

在构造时，`pybind11`将Python回调函数的句柄作为`std::function`对象传递。

需要特别提及的是：
```cpp
/*
 * NOTE
 * ==========
 * pybind11 will invoke a copy if this is `JacobianVector &`,
 * and modifications in Python will not be reflected.
 *
 * This is safe because this is passing a const pointer,
 * and pybind11 will maintain the `std::vector` memory layout.
 * Thus the pointer will never be invalidated.
 */
using CustomErrorFunction = std::function<Vector(const CustomFactor&, const Values&, const JacobianVector*)>;
```
这在`pybind11`文档中没有记录。如果要在Python-C++边界上实现类似的"可变"参数，需要注意这一点。

## DoglegOptimizer
# DoglegOptimizer

## 概述

GTSAM中的`DoglegOptimizer`类是一种专门的优化算法，设计用于求解非线性最小二乘问题。它实现了Dogleg方法，这是一种结合最速下降和高斯-牛顿方法的混合方法。

Dogleg方法的特点在于使用两个不同的步骤：

1. **Cauchy点(Cauchy Point)**：最速下降方向，计算为：
   $$ p_u = -\alpha \nabla f(x) $$
   其中$\alpha$是标量步长。

2. **高斯-牛顿步(Gauss-Newton Step)**：线性化问题的解，提供更精确但计算成本更高的步长：
   $$ p_{gn} = -(J^T J)^{-1} J^T r $$
   其中$J$是Jacobian矩阵，$r$是残差向量。

Dogleg步$p_{dl}$是这两个步的组合，由信赖域半径$\Delta$确定。

主要特性：

- **混合方法**：结合了最速下降和高斯-牛顿两种方法的优点。
- **信赖域方法**：利用信赖域确定步长，在高斯-牛顿的精度和最速下降的鲁棒性之间取得平衡。
- **非线性问题高效**：设计用于有效处理复杂的非线性最小二乘问题。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/DoglegOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 关键方法

请参见基类[NonlinearOptimizer.ipynb](NonlinearOptimizer.ipynb)。

## 参数

`DoglegParams`类定义了Powell's Dogleg优化算法特定的参数：

| 参数 | 描述 |
|-----------|-------------|
| `deltaInitial` | 初始信赖域半径，控制步长（默认: 1.0） |
| `verbosityDL` | 控制算法特定的诊断输出（选项: SILENT, VERBOSE） |

这些参数补充了从`NonlinearOptimizerParams`继承的标准优化参数，包括：

- 最大迭代次数
- 相对和绝对误差阈值
- 误差函数详细程度
- 线性求解器类型

Powell的Dogleg算法在信赖域框架内结合了高斯-牛顿和梯度下降方法。`deltaInitial`参数定义此信赖域的初始大小，该大小在优化过程中根据线性近似与非线性函数的匹配程度自适应变化。

## 使用注意事项

- **初始猜测(Initial Guess)**：Dogleg优化器的性能可能对初始猜测敏感。好的初始估计可以显著加速收敛。
- **参数调优**：初始信赖域半径和其他参数的选择会影响收敛速率和优化的稳定性。

## 文件

- [DoglegOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/DoglegOptimizer.h)
- [DoglegOptimizer.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/DoglegOptimizer.cpp)
- [DoglegParams.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/DoglegParams.h)

## ExpressionFactor
# ExpressionFactor

## 概述

GTSAM中的`ExpressionFactor`类是一个模板类，设计用于在非线性优化的上下文中与因子图配合使用。它表示一个可以从[GTSAM expression](../../../doc/expressions.md)构造的因子，允许在优化问题中灵活高效地计算误差项。

`ExpressionFactor`类允许用户在C++中基于表达式定义因子，使用（反向）自动微分来计算其Jacobian。

## 主要方法

### 构造函数

`ExpressionFactor`类提供了一个构造函数，允许用特定的表达式和测量初始化因子：

```cpp
  /**
   * Constructor: creates a factor from a measurement and measurement function
   *   @param noiseModel the noise model associated with a measurement
   *   @param measurement actual value of the measurement, of type T
   *   @param expression predicts the measurement from Values
   * The keys associated with the factor, returned by keys(), are sorted.
   */
  ExpressionFactor(const SharedNoiseModel& noiseModel,  //
                   const T& measurement, const Expression<T>& expression)
      : NoiseModelFactor(noiseModel), measured_(measurement) {
    initialize(expression);
  }
```

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ExpressionFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## ExpressionFactorGraph
# ExpressionFactorGraph

## 概述

GTSAM中的`ExpressionFactorGraph`类是一种专门设计用于与表达式配合使用的因子图。它通过允许从实现自动微分的[GTSAM expressions](../../../doc/expressions.md)创建因子来扩展标准因子图的能力。它创建[ExpressionFactors](ExpressionFactor.ipynb)。

### 添加表达式因子

使用**addExpressionFactor**：此方法允许用户基于符号表达式向图添加新因子。表达式定义了因子涉及的变量之间的关系。
```c++
  template<typename T>
  void addExpressionFactor(const Expression<T>& h, const T& z,
      const SharedNoiseModel& R) {
    using F = ExpressionFactor<T>;
    push_back(std::allocate_shared<F>(Eigen::aligned_allocator<F>(), R, z, h));
  }
```

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ExpressionFactorGraph.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## ExtendedKalmanFilter
# ExtendedKalmanFilter

## 概述

GTSAM中的`ExtendedKalmanFilter`类是[扩展卡尔曼滤波器(EKF)](https://en.wikipedia.org/wiki/Extended_Kalman_filter)的实现，这是估计非线性动态系统状态的强大工具。

另请参见[此笔记本](../../../python/gtsam/examples/easyPoint2KalmanFilter.ipynb)了解以下C++示例的Python版本。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ExtendedKalmanFilter.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 使用ExtendedKalmanFilter类

GTSAM中的`ExtendedKalmanFilter`类提供了一种使用因子图实现卡尔曼滤波的灵活方法。以下是基于[easyPoint2KalmanFilter.cpp](https://github.com/borglab/gtsam/blob/develop/examples/easyPoint2KalmanFilter.cpp)中的示例的分步指南：

### 使用ExtendedKalmanFilter的步骤

1. **初始化滤波器**：
   - 定义初始状态（例如位置）及其协方差。
   - 为初始状态创建键。
   - 使用初始状态和协方差实例化`ExtendedKalmanFilter`对象。

   ```cpp
   Point2 x_initial(0.0, 0.0);
   SharedDiagonal P_initial = noiseModel::Diagonal::Sigmas(Vector2(0.1, 0.1));
   Symbol x0('x', 0);
   ExtendedKalmanFilter<Point2> ekf(x0, x_initial, P_initial);
   ```

2. 预测下一状态：
   - 使用BetweenFactor定义运动模型。
   - 使用predict方法预测下一状态。
   ```cpp
   Symbol x1('x', 1);
   Point2 difference(1, 0);
   SharedDiagonal Q = noiseModel::Diagonal::Sigmas(Vector2(0.1, 0.1), true);
   BetweenFactor<Point2> factor1(x0, x1, difference, Q);
   Point2 x1_predict = ekf.predict(factor1);
   ```

3. 使用测量更新状态：
   - 使用PriorFactor定义测量模型。
   - 使用update方法更新状态。
   ```cpp
   Point2 z1(1.0, 0.0);
   SharedDiagonal R = noiseModel::Diagonal::Sigmas(Vector2(0.25, 0.25), true);
   PriorFactor<Point2> factor2(x1, z1, R);
   Point2 x1_update = ekf.update(factor2);
   ```
4. 对后续时间步重复：
   - 对后续状态和测量重复预测和更新步骤。

## 示例用例
此示例演示了使用简单线性运动模型和位置测量跟踪移动的2D点。`ExtendedKalmanFilter`类允许使用GTSAM的因子图框架灵活地对运动和测量过程进行建模。

完整实现请参见[easyPoint2KalmanFilter.cpp](https://github.com/borglab/gtsam/blob/develop/examples/easyPoint2KalmanFilter.cpp)文件。

## ExtendedPriorFactor
# ExtendedPriorFactor

## 用途和受众

`ExtendedPriorFactor`是[PriorFactor](PriorFactor.ipynb)和其他软锚定机制底层的通用构建块。它允许您在任意流形值类型的切空间中表达一个（可选平移的）高斯（或鲁棒）似然。本笔记面向设计自定义因子、尝试非零切空间均值或使用非传统流形类型的高级GTSAM用户。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ExtendedPriorFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
# Install GTSAM and Plotly from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass # Not in Colab
```

## 数学定义

给定值类型`T`（李群或一般流形），具有局部坐标算子`Local(x, y)`（从流形映射到`x`处的切空间），一个原点值`o`，一个可选的切空间均值向量`m`（默认0），以及一个带有（鲁棒）损失`rho`的噪声模型，该因子定义：

- 原始误差：`e(x) = - Local(x, o) - m`（如果提供了`m`，否则为`-Local(x,o)`）
- 损失：`loss(x) = rho( || e(x) ||^2_Sigma )`其中`Sigma`是噪声模型内部的协方差。
- 似然：`L(x) = exp( - loss(x) )`。

对于李群，其中`Local(x,o) = Log( x^{-1} o )`，这是（在指数上差一个加性常数）扩展集中高斯：
$$  e(x) = - (\operatorname{Log}(x^{-1} o) + m) $$

选择非零均值`m`有效地将最大似然点从`o`移动到`o.retract(m)`。

## 与PriorFactor的关系

`PriorFactor<T>`是一个特化，将`m = 0`并将原点命名为`prior`。它还通过使用`-Local(x, prior)`形式将Jacobian实现为单位矩阵，避免了计算`Local`的导数。

使用`PriorFactor`，除非您明确需要平移的切空间均值或想要将结果分布视为似然（例如，用于概率加权或退火策略）。

## 何时直接使用ExtendedPriorFactor

- 您需要以`origin.retract(mean)`为中心的软约束，但希望出于数值原因在`origin`处保持固定的线性化框架。
- 您正在实现一个概念上对流形变量上的潜在密度建模的因子（例如，从学习模型产生的均偏移向量的位姿先验）。
- 您想要在候选值上求值归一化或相对似然（`likelihood(x)`便捷方法）。
- 您使用鲁棒噪声模型来降低伪先验切空间残差中的离群值权重。
- 您原型化新的流形类型：避免提供局部坐标的分析Jacobian（超出单位矩阵）可以简化早期实验。

## C++ API回顾

构造函数（以`VALUE`为模板参数）：

```cpp
ExtendedPriorFactor(Key key, const VALUE& origin,
                    const SharedNoiseModel& model,
                    const std::optional<Vector>& mean = {});
```

关键方法：

- `Vector evaluateError(const VALUE& x, OptionalMatrixType H)`返回切空间残差（如果`H`则为单位Jacobian）。
- `double error(const VALUE& x)`负对数似然（鲁棒）。
- `double likelihood(const VALUE& x)`返回`exp(-error(x))`。
- 访问器：`origin()`, `mean()`。

## 实用示例 (Pose2)

我们使用Python绑定复现单元测试的核心逻辑（参见`nonlinear/tests/testExtendedPriorFactor.cpp`）。

```python
# Example: Pose2 ExtendedPriorFactor usage
import math
import numpy as np
import gtsam
from gtsam import Pose2, noiseModel

key = 1
origin = Pose2(1.0, 2.0, 0.3)
model = noiseModel.Isotropic.Sigma(3, 0.5)  # std dev 0.5 in all dims
mean = np.array([0.1, 0.2, 0.05])  # tangent space shift

factor = gtsam.ExtendedPriorFactorPose2(key, origin, mean, model)
# Evaluate at origin.retract(mean): should yield near-zero residual and likelihood ~ 1
x_mode = origin.retract(mean)
residual_at_mode = factor.evaluateError(x_mode)
likelihood_at_mode = factor.likelihood(x_mode)
residual_at_mode, likelihood_at_mode
```

### 解释残差

因为报告的Jacobian是单位矩阵，该因子在数值上稳定且廉价：线性化仅贡献单位项。残差是局部坐标中的有符号差，由均值平移。

## 添加到图中

您可以直接将`ExtendedPriorFactor`添加到`NonlinearFactorGraph`中，但与`PriorFactor`不同，没有便捷的`addPrior<T>()`方法。您需要显式构造和添加。

```python
from gtsam import NonlinearFactorGraph, Values, LevenbergMarquardtOptimizer

graph = NonlinearFactorGraph()
graph.push_back(factor)  # our ExtendedPriorFactor

initial = Values()
# Provide an initial guess somewhat away from the mode
initial.insert(key, Pose2(1.2, 1.9, 0.25))

params = gtsam.LevenbergMarquardtParams()
optimizer = LevenbergMarquardtOptimizer(graph, initial, params)
result = optimizer.optimize()
final_pose = result.atPose2(key)
final_pose
```

## 设计说明和陷阱

1. 切空间的框架固定在`origin`处。非常大的均值可能隐式请求远离线性化区域的运动；如果优化困难，考虑重新中心化或使用不同的建模方法。
2. 鲁棒模型（例如`noiseModel.Robust.Create(mEstimator, baseModel)`）将其损失应用于残差的平方马氏距离；这让您可以将切残差视为容易受离群值影响的数量。
3. 对于自定义流形类型，确保`traits<T>::Local`和`retract`（如果您想计算*mode*值，隐式通过`retract`）已实现且一致。
4. 与某些先验不同，该因子有意不存储或重新计算动态Jacobian；单位矩阵假设是一个有意的设计选择。
5. 序列化支持依赖`NoiseModelFactor1`以实现向后兼容；如果扩展该类，请记住这一点。

## 何时不使用

- 如果您只需要零均值软先验：更喜欢[PriorFactor](PriorFactor.ipynb)。
- 如果您需要硬等式约束：考虑`NonlinearEquality`。
- 如果残差应该依赖于另一个变量或测量，请设计自定义因子，而不是将复杂性嵌入`Local`。

## 另请参见

- [PriorFactor](PriorFactor.ipynb)
- 其他流形先验：`BetweenFactor`, `NonlinearEquality`及鲁棒变体
- 底层噪声模型：`noiseModel::Isotropic`, `noiseModel::Diagonal`, 鲁棒创建器

## FixedLagSmoother
# FixedLagSmoother

## 概述

`FixedLagSmoother`类是[BatchFixedLagSmoother](BatchFixedLagSmoother.ipynb)和[IncrementalFixedLagSmoother](IncrementalFixedLagSmoother.ipynb)的基类。

它提供了一个用于非线性因子图中固定滞后平滑的API。它维护最近变量的滑动窗口，并边缘化掉更旧的变量。这在内存和计算效率至关重要的实时应用中特别有用。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/FixedLagSmoother.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 数学公式

在固定滞后平滑中，目标是在给定直到时间$t$的所有测量下估计状态$\mathbf{x}_t$，但仅保留最近状态的固定窗口。优化问题可以表达为：
$$
\min_{\mathbf{x}_{t-L:t}} \sum_{i=1}^{N} \| \mathbf{h}_i(\mathbf{x}_{t-L:t}) - \mathbf{z}_i \|^2
$$
其中$L$是固定滞后，$\mathbf{h}_i$是测量函数，$\mathbf{z}_i$是测量值。
在实践中，函数$\mathbf{h}_i$仅依赖于状态变量$\mathbf{X}_i$的子集，优化是在一组$N$个*因子*$\phi_i$上执行的：
$$
\min_{\mathbf{x}_{t-L:t}} \sum_{i=1}^{N} \| \phi_i(\mathbf{X}_i; \mathbf{z}_i) \|^2
$$
下面的API允许用户在每次迭代中添加新因子，这些因子在不依赖于滞后中的任何变量后会被自动修剪。

## GaussNewtonOptimizer 高斯-牛顿优化器
# GaussNewtonOptimizer

## 概述

GTSAM中的`GaussNewtonOptimizer`类旨在使用高斯-牛顿算法优化非线性因子图。此类特别适用于代价函数在最小值附近可以被二次函数很好地近似的问题。高斯-牛顿方法是一种迭代优化技术，通过在每次迭代中线性化非线性系统来更新解。

高斯-牛顿算法基于在当前估计$x_k$周围线性化非线性残差$r(x)$的思想。更新步从求解法方程导出：

$$ J(x_k)^T J(x_k) \Delta x = -J(x_k)^T r(x_k) $$

其中$J(x_k)$是残差关于变量的Jacobian。解$\Delta x$用于更新估计：

$$ x_{k+1} = x_k + \Delta x $$

此过程迭代重复直到收敛。

主要特性：

- **迭代优化**：优化器通过在当前估计周围线性化非线性系统来迭代细化解。
- **收敛控制**：通过参数（如最大迭代次数和相对误差容差）提供控制收敛的机制。
- **与GTSAM集成**：与GTSAM的因子图框架无缝集成，允许与各种类型的因子和变量一起使用。

## 关键方法

请参见基类[NonlinearOptimizer.ipynb](NonlinearOptimizer.ipynb)。

## 参数

高斯-牛顿优化器使用从`NonlinearOptimizerParams`继承的标准优化参数，包括：

- 最大迭代次数
- 相对和绝对误差阈值
- 误差函数详细程度
- 线性求解器类型

## 使用注意事项

- **初始猜测**：初始猜测的质量可以显著影响高斯-牛顿优化器的收敛性和性能。
- **非凸性**：由于该方法依赖线性近似，对于高度非凸问题或初始估计较差的问题可能表现不佳。
- **性能**：对于在解附近能被二次模型很好近似的问题，高斯-牛顿方法通常比其他非线性优化方法（如Levenberg-Marquardt）更快。

## 文件

- [GaussNewtonOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GaussNewtonOptimizer.h)
- [GaussNewtonOptimizer.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GaussNewtonOptimizer.cpp)

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/GaussNewtonOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## GncOptimizer
# GncOptimizer

## 概述

GTSAM中的`GncOptimizer`类设计用于使用渐进非凸(GNC, Graduated Non-Convexity)执行鲁棒优化。此方法在优化问题受离群值影响的情况下特别有用。GNC方法逐渐从问题的凸近似过渡到原始的非凸问题，从而提高鲁棒性和收敛性。

`GncOptimizer`利用鲁棒代价函数$\rho(e)$，其中$e$是误差项。目标是最小化所有测量上的这些鲁棒代价之和：

$$
\min_x \sum_i \rho(e_i(x))
$$

在GNC的上下文中，鲁棒代价函数逐渐从凸近似变换为原始非凸形式。此变换由参数$\mu$控制，该参数在优化过程中调整：

$$
\rho_\mu(e) = \frac{1}{\mu} \rho(\mu e)
$$

随着$\mu$增加，函数$\rho_\mu(e)$从凸形过渡到非凸形，使优化器能够有效地处理离群值。

主要特性：

- **鲁棒优化**：GncOptimizer专门设计用于处理带有离群值的优化问题，使用可以减轻其影响的鲁棒代价函数。
- **渐进非凸**：此技术允许优化器从凸问题开始，逐渐将其变换为原始的非凸问题，有助于避免局部最小值。
- **可自定义参数**：用户可以调整各种参数来控制优化器的行为，例如鲁棒损失函数的类型和控制GNC过程的参数。

## 关键方法

请参见基类[NonlinearOptimizer.ipynb](NonlinearOptimizer.ipynb)。

## 参数

`GncParams`类定义了GNC优化算法特定的参数：

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------------|-------------|
| lossType | GncLossType | TLS | 鲁棒损失函数类型（GM = Geman McClure 或 TLS = 截断最小二乘） |
| maxIterations | size_t | 100 | 最大迭代次数 |
| muStep | double | 1.4 | GNC中减小/增加mu的乘法因子 |
| relativeCostTol | double | 1e-5 | 停止迭代的相对代价变化阈值 |
| weightsTol | double | 1e-4 | 停止迭代的权重接近二值的阈值（仅TLS） |
| verbosity | Verbosity enum | SILENT | 详细程度级别（选项: SILENT, SUMMARY, MU, WEIGHTS, VALUES） |
| knownInliers | IndexVector | Empty | 因子图中已知为内点(inlier)的测量槽位 |
| knownOutliers | IndexVector | Empty | 因子图中已知为离群值(outlier)的测量槽位 |

这些参数补充了从`NonlinearOptimizerParams`继承的标准优化参数。

## 使用注意事项

- **离群值拒绝**：`GncOptimizer`在存在大量离群值的场景中特别有效，例如SLAM或光束法平差问题。
- **参数调优**：正确调整GNC参数对实现最佳性能至关重要。用户应尝试不同的设置以找到适合其特定问题的最佳配置。

## 文件

- [GncOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GncOptimizer.h)
- [GncParams.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/GncParams.h)

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/GncOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## IncrementalFixedLagSmoother
# IncrementalFixedLagSmoother

## 概述

`IncrementalFixedLagSmoother`是一个使用[iSAM2](iSAM2.ipynb)进行增量推断的[FixedLagSmoother](FixedLagSmoother.ipynb)。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/IncrementalFixedLagSmoother.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## ISAM2
# ISAM2

## 概述

GTSAM中的`ISAM2`类是一个增量平滑与建图(Incremental Smoothing and Mapping)算法，在添加新测量时高效更新非线性优化问题的解。此类在SLAM（同时定位与建图）等实时性能至关重要的应用中特别有用。

该算法在{cite:t}`http://dx.doi.org/10.1177/0278364911430419`的2012年IJRR论文中描述。背景知识另请参见{cite:t}`https://doi.org/10.1561/2300000043`的最新小册子。

本笔记专注Python `ISAM2` API，包括暴露当前线性化高斯结构的继承贝叶斯树查询方法。

## 主要特性

- **增量更新**：`ISAM2`允许对因子图进行增量更新，避免了每次新测量都需要从零开始求解整个问题。
- **非线性优化**：能够处理非线性系统，利用迭代优化技术精细化估计。
- **高效变量重排序**：动态重新排序变量以保持稀疏性并提高计算效率。
- **贝叶斯树查询**：通过镜像`GaussianBayesTree`接口的边缘、联合和贝叶斯网查询暴露当前线性化高斯。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/ISAM2.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2026, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

```python
import graphviz
from IPython.display import display
import numpy as np
import gtsam
from gtsam.symbol_shorthand import X
```

## 主要方法

### 初始化和配置

- **ISAM2构造函数**：初始化`ISAM2`对象，带有配置算法行为的可选参数，如重线性化阈值和排序策略。
- **size / empty**：报告增量更新后当前的贝叶斯树大小。

### 更新图

- **update**：将新因子和变量纳入现有因子图中。此方法执行核心增量更新，基于新测量细化解。

### 获取结果

- **calculateEstimate**：检索因子图中变量的当前估计。此方法可以调用特定变量键来获取其估计。
- **marginalFactor / marginalInformation / marginalCovariance**：查询单个变量的当前线性化高斯边缘分布。
- **jointMarginalInformation / jointMarginalCovariance**：为键集合物化稠密联合边缘分布。

### 继承的贝叶斯树查询

- **joint**：提取仅支持查询变量的约化高斯因子图。
- **jointBayesNet**：提取相应的约化高斯贝叶斯网。
- **numCachedSeparatorMarginals / deleteCachedShortcuts**：检查和清除由贝叶斯树边缘查询创建的快捷方式缓存。

### 高级特性

- **getFactorsUnsafe**：提供对内部因子图的访问，允许高级操作和自定义分析。

## 示例

我们将增量构建一个小型的`Pose2`链。

### 初始化和配置

```python
prior_noise = gtsam.noiseModel.Diagonal.Sigmas(np.array([0.2, 0.2, 0.1]))
odom_noise = gtsam.noiseModel.Diagonal.Sigmas(np.array([0.1, 0.1, 0.05]))

params = gtsam.ISAM2Params()
isam = gtsam.ISAM2(params)
```

### 更新图

以下我们执行两次更新。

```python
new_factors = gtsam.NonlinearFactorGraph()
new_values = gtsam.Values()
new_factors.addPriorPose2(X(0), gtsam.Pose2(0.0, 0.0, 0.0), prior_noise)
new_values.insert(X(0), gtsam.Pose2(0.0, 0.0, 0.0))
isam.update(new_factors, new_values)

new_factors = gtsam.NonlinearFactorGraph()
new_values = gtsam.Values()
new_factors.add(gtsam.BetweenFactorPose2(X(0), X(1), gtsam.Pose2(1.0, 0.0, 0.0), odom_noise))
new_factors.add(gtsam.BetweenFactorPose2(X(1), X(2), gtsam.Pose2(1.0, 0.0, 0.0), odom_noise))
new_values.insert(X(1), gtsam.Pose2(1.0, 0.1, 0.02))
new_values.insert(X(2), gtsam.Pose2(2.0, -0.1, -0.01))
isam.update(new_factors, new_values)
```

### 可视化贝叶斯树

由于`ISAM2`继承了贝叶斯树可视化方法，我们可以用Graphviz直接显示当前树结构。

```python
display(graphviz.Source(isam.dot()))
```

ISAM2内部的贝叶斯树随着更新的纳入而增长。继承的`size()`和`empty()`方法是方便快速的健全检查。

```python
print("empty:", isam.empty())
print("number of cliques:", isam.size())
```

### 获取结果

`calculateEstimate`检索所有当前的非线性估计。如果有很多变量，这可能很昂贵。

```python

estimate = isam.calculateEstimate()
estimate
```

### 获取单变量结果

对于单个键，`ISAM2`暴露了模板化的`calculateEstimate`。我们还可以检查`marginalFactor`返回的条件概率，以及相应的稠密信息和协方差矩阵。这些对于贝叶斯树表示的当前线性化高斯的精确值。

```python
print("x1 estimate:", isam.calculateEstimatePose2(X(1)))

x1_factor = isam.marginalFactor(X(1))
x1_information = isam.marginalInformation(X(1))
x1_covariance = isam.marginalCovariance(X(1))

print(type(x1_factor).__name__)
print("information shape:", x1_information.shape)
print("covariance shape:", x1_covariance.shape)
np.round(x1_covariance, 4)
```

### 获取联合边缘分布

可以直接请求稠密联合信息和协方差。返回的`JointMarginal`提供完整矩阵和键控块访问。

```python
query_keys = [X(2), X(0)]
joint_information = isam.jointMarginalInformation(query_keys)
joint_covariance = isam.jointMarginalCovariance(query_keys)

print("joint information shape:", joint_information.fullMatrix().shape)
print("cross block shape:", joint_covariance.at(X(2), X(0)).shape)
np.round(joint_covariance.fullMatrix(), 4)
```

### 继承的贝叶斯树查询方法

继承的`joint`和`jointBayesNet`方法暴露了底层的约化高斯系统。双键重载对成对查询很有用，而`KeyVector`重载推广到更大的集合。

```python
pair_joint_graph = isam.joint(X(0), X(2))
multi_joint_graph = isam.joint([X(2), X(1), X(0)])
pair_joint_bayes_net = isam.jointBayesNet(X(0), X(2))
multi_joint_bayes_net = isam.jointBayesNet([X(2), X(1), X(0)])

print("pair joint graph factors:", pair_joint_graph.size())
print("multi joint graph factors:", multi_joint_graph.size())
print("pair joint Bayes net conditionals:", pair_joint_bayes_net.size())
print("multi joint Bayes net conditionals:", multi_joint_bayes_net.size())
```

### 高级特性

基于快捷方式的贝叶斯树查询会填充缓存的分隔符边缘分布。缓存计数器有助于计时重复的边缘查询，`deleteCachedShortcuts()`清除该状态。

```python
before = isam.numCachedSeparatorMarginals()
_ = isam.jointBayesNet([X(0), X(2)])
after_query = isam.numCachedSeparatorMarginals()
isam.deleteCachedShortcuts()
after_clear = isam.numCachedSeparatorMarginals()

print("before:", before)
print("after query:", after_query)
print("after clear:", after_clear)
```

其他检查方法暴露了当前线性化的非线性问题和贝叶斯树状态。这些在比较`ISAM2`查询与批量计算时很有用。

```python
linearization_point = isam.getLinearizationPoint()
linearized_factors = isam.getFactorsUnsafe()
print("variables in linearization point:", linearization_point.size())
print("linearized factor count:", linearized_factors.size())
```

## LevenbergMarquardtOptimizer 列文伯格-马夸尔特优化器
# LevenbergMarquardtOptimizer

## 概述

GTSAM中的`LevenbergMarquardtOptimizer`类是一个专门的优化器，实现了列文伯格-马夸尔特(Levenberg-Marquardt)算法。此算法是求解非线性最小二乘问题的流行选择，在各种应用中广泛使用，如计算机视觉、机器人和机器学习。

列文伯格-马夸尔特算法是一种迭代技术，在高斯-牛顿算法和梯度下降法之间进行插值。它对于优化解预期在初始猜测附近的问题特别有用。

列文伯格-马夸尔特算法寻求最小化以下形式的代价函数$F(x)$：

$$
F(x) = \frac{1}{2} \sum_{i=1}^{m} r_i(x)^2
$$

其中$r_i(x)$是残差。算法的更新规则为：

$$
x_{k+1} = x_k - (J^T J + \lambda I)^{-1} J^T r
$$

这里$J$是残差的Jacobian矩阵，$\lambda$是阻尼参数，$I$是单位矩阵。

主要特性：

- **非线性优化**：该类设计用于高效处理非线性优化问题。
- **阻尼机制**：它包含阻尼参数以控制步长，在高斯-牛顿和梯度下降方法之间取得平衡。
- **迭代改进**：优化器迭代地细化解，在每一步减少误差。

## 关键方法

请参见基类[NonlinearOptimizer.ipynb](NonlinearOptimizer.ipynb)。

## 参数

`LevenbergMarquardtParams`类定义了此优化算法特定的参数：

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------------|-------------|
| lambdaInitial | double | 1e-5 | 初始LM阻尼项 |
| lambdaFactor | double | 10.0 | 调整lambda时乘以或除以的量 |
| lambdaUpperBound | double | 1e5 | 在假定优化失败前尝试的最大lambda |
| lambdaLowerBound | double | 0.0 | LM中使用的最小lambda |
| verbosityLM | VerbosityLM | SILENT | LM的详细程度级别 |
| minModelFidelity | double | 1e-3 | 接受LM迭代结果的modelFidelity下界 |
| logFile | std::string | "" | 可选的CSV日志文件，格式为[iteration, time, error, lambda] |
| diagonalDamping | bool | false | 如果为true，使用Hessian的对角线 |
| useFixedLambdaFactor | bool | true | 如果为true，根据lambdaFactor应用常数增加（或减少） |
| minDiagonal | double | 1e-6 | 使用对角阻尼时饱和最小对角条目 |
| maxDiagonal | double | 1e32 | 使用对角阻尼时饱和最大对角条目 |

## 使用注意事项

- 初始猜测的选择可以显著影响收敛速度和解决方案的质量。
- 正确调整阻尼参数$\lambda$对平衡收敛速率和稳定性至关重要。
- 当残差在解附近近似线性时，优化器最有效。

## 文件

- [LevenbergMarquardtOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/LevenbergMarquardtOptimizer.h)
- [LevenbergMarquardtOptimizer.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/LevenbergMarquardtOptimizer.cpp)
- [LevenbergMarquardtParams.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/LevenbergMarquardtParams.h)
- [LevenbergMarquardtParams.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/LevenbergMarquardtParams.cpp)

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/LevenbergMarquardtOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## LinearContainerFactor
# LinearContainerFactor

## 概述

GTSAM中的`LinearContainerFactor`类是一个专门的因子，将线性因子封装在非线性因子图中。这在边缘化变量时广泛使用。

## 主要特性

- **线性因子的封装**：`LinearContainerFactor`的主要功能是存储一个线性因子及其关联的值，使其能够在非线性上下文中使用。
- **误差计算**：它提供在给定一组值下计算因子误差的机制。
- **Jacobian计算**：该类可以计算Jacobian矩阵，这对优化过程至关重要。

## 关键方法

- **LinearContainerFactor**：此构造函数使用线性因子并可选用值初始化`LinearContainerFactor`。它作为创建此类实例的入口点。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/LinearContainerFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## Marginals 边缘分布
# Marginals

## 概述

GTSAM中的`Marginals`类从在选定线性化点周围的非线性或线性因子图计算高斯边缘分布。在常见的非线性情况下，通常的模式是：

1. 构建一个`NonlinearFactorGraph`，
2. 优化它以获得解`Values`，以及
3. 构造`Marginals(graph, result)`来查询不确定性。

最常见的查询是单变量边缘协方差、单变量边缘信息以及一小部分变量的联合协方差或信息。在内部，GTSAM使用高斯贝叶斯树来高效回答这些查询，但用户通常只需直接调用`Marginals`方法。

关于贝叶斯树协方差恢复的算法故事，参见博客文章[Fast Covariance Recovery in GTSAM Bayes Trees](https://gtsam.org/2026/03/29/bayes-tree-covariance-recovery.html)。完整技术细节，参见[CovarianceRecovery.pdf](https://github.com/borglab/gtsam/blob/develop/doc/CovarianceRecovery.pdf)。

GTSAM Copyright 2010-2026, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/Marginals.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

```python
import numpy as np
import gtsam
from gtsam.symbol_shorthand import L, X
```

## Marginals返回什么

主要方法有：

- `marginalCovariance(key)`: 单个变量的稠密协方差矩阵
- `marginalInformation(key)`: 单个变量的稠密信息矩阵
- `jointMarginalCovariance(keys)`: 包含多个变量联合协方差的`JointMarginal`
- `jointMarginalInformation(keys)`: 包含多个变量联合信息矩阵的`JointMarginal`

`JointMarginal`允许您使用`fullMatrix()`访问完整稠密矩阵，或使用`at(key_i, key_j)`请求特定的块。在实践中，`at(...)`是检索块的最安全方式，因为它避免依赖内部的块布局。

## 小型平面SLAM示例

以下示例创建一个包含三个`Pose2`变量和两个`Point2`路标的微型非线性SLAM问题。优化图后，我们使用`Marginals`恢复几种不同的不确定性查询。

```python
def create_planar_slam_problem():
    graph = gtsam.NonlinearFactorGraph()

    x1 = X(1)
    x2 = X(2)
    x3 = X(3)
    l1 = L(1)
    l2 = L(2)

    prior_noise = gtsam.noiseModel.Diagonal.Sigmas(np.array([0.3, 0.3, 0.1]))
    odometry_noise = gtsam.noiseModel.Diagonal.Sigmas(np.array([0.2, 0.2, 0.1]))
    measurement_noise = gtsam.noiseModel.Diagonal.Sigmas(np.array([0.1, 0.2]))

    graph.addPriorPose2(x1, gtsam.Pose2(0.0, 0.0, 0.0), prior_noise)
    graph.add(gtsam.BetweenFactorPose2(x1, x2, gtsam.Pose2(2.0, 0.0, 0.0), odometry_noise))
    graph.add(gtsam.BetweenFactorPose2(x2, x3, gtsam.Pose2(2.0, 0.0, 0.0), odometry_noise))

    graph.add(gtsam.BearingRangeFactor2D(x1, l1, gtsam.Rot2.fromDegrees(45), np.sqrt(8.0), measurement_noise))
    graph.add(gtsam.BearingRangeFactor2D(x2, l1, gtsam.Rot2.fromDegrees(90), 2.0, measurement_noise))
    graph.add(gtsam.BearingRangeFactor2D(x3, l2, gtsam.Rot2.fromDegrees(90), 2.0, measurement_noise))

    initial = gtsam.Values()
    initial.insert(x1, gtsam.Pose2(0.1, -0.1, 0.05))
    initial.insert(x2, gtsam.Pose2(2.1, 0.1, -0.02))
    initial.insert(x3, gtsam.Pose2(4.2, -0.1, 0.04))
    initial.insert(l1, gtsam.Point2(1.8, 2.2))
    initial.insert(l2, gtsam.Point2(4.2, 2.1))

    return graph, initial, (x1, x2, x3, l1, l2)
```

```python
graph, initial, (x1, x2, x3, l1, l2) = create_planar_slam_problem()

params = gtsam.LevenbergMarquardtParams()
optimizer = gtsam.LevenbergMarquardtOptimizer(graph, initial, params)
result = optimizer.optimize()

marginals = gtsam.Marginals(graph, result)
```

## 单变量查询

对于单个变量，`Marginals`可以返回协方差矩阵或信息矩阵。

```python
pose2_covariance = marginals.marginalCovariance(x2)
pose2_information = marginals.marginalInformation(x2)

print("Covariance of x2:\n", np.round(pose2_covariance, 4))
print("Information of x2:\n", np.round(pose2_information, 4))
```

## 联合查询

对于多个变量，`Marginals`返回一个`JointMarginal`。您可以检查完整稠密矩阵，或按键检索个别协方差块。

```python
joint_covariance = marginals.jointMarginalCovariance([x2, l1, x3])
joint_information = marginals.jointMarginalInformation([x2, l1, x3])

print("Full joint covariance:\n", np.round(joint_covariance.fullMatrix(), 4))
print("Covariance block Cov[x2, l1]:\n", np.round(joint_covariance.at(x2, l1), 4))
print("Information block Info[x3, x3]:\n", np.round(joint_information.at(x3, x3), 4))
```

## 性能指导

使用`Marginals`时用户通常不需要直接管理贝叶斯树。GTSAM在内部构建或重用高斯贝叶斯树，并在回答协方差查询时利用其结构。

在实践中，这意味着：

- 单变量查询已经非常高效，
- 双变量联合查询也已经局部化且高效，
- 更大的联合查询现在比以前快得多，因为内部支持提取更有选择性。

所以面向用户的建议很简单：调用您实际需要的查询，让`Marginals`管理贝叶斯树的细节。

## 另请参见

- 博客文章: [Fast Covariance Recovery in GTSAM Bayes Trees](https://gtsam.org/2026/03/29/bayes-tree-covariance-recovery.html)
- 论文: [CovarianceRecovery.pdf](https://github.com/borglab/gtsam/blob/develop/doc/CovarianceRecovery.pdf)
- 头文件: [`Marginals.h`](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/Marginals.h)

## NonlinearConjugateGradientOptimizer 非线性共轭梯度优化器
# NonlinearConjugateGradientOptimizer

## 概述

GTSAM中的`NonlinearConjugateGradientOptimizer`类是用于优化非线性函数的非线性共轭梯度法的实现。此优化器对于求解Hessian矩阵不容易计算或存储的大规模优化问题特别有用。共轭梯度法是一种迭代算法，通过沿一系列共轭方向搜索寻找函数的最小值。

非线性共轭梯度法通过迭代更新解$x_k$来最小化非线性函数$f(x)$：

$$ x_{k+1} = x_k + \alpha_k p_k $$

其中$p_k$是搜索方向，$\alpha_k$是由线搜索确定的步长。搜索方向$p_k$使用函数的梯度和共轭梯度更新公式（如Fletcher-Reeves或Polak-Ribiere公式）计算：

- **Fletcher-Reeves**：
  $$ \beta_k^{FR} = \frac{\nabla f(x_{k+1})^T \nabla f(x_{k+1})}{\nabla f(x_k)^T \nabla f(x_k)} $$

- **Polak-Ribiere**：
  $$ \beta_k^{PR} = \frac{\nabla f(x_{k+1})^T (\nabla f(x_{k+1}) - \nabla f(x_k))}{\nabla f(x_k)^T \nabla f(x_k)} $$

$\beta_k$的选择影响算法的收敛性质。

主要特性：

- **优化方法**：实现非线性共轭梯度法，这是线性共轭梯度法到非线性优化问题的扩展。
- **效率**：由于其迭代性质和与需要Hessian矩阵的方法相比减少的内存需求，适合大规模问题。
- **灵活性**：可以与各种线搜索策略和共轭梯度更新公式一起使用。

## 关键方法

请参见基类[NonlinearOptimizer.ipynb](NonlinearOptimizer.ipynb)。

## 参数

非线性共轭梯度优化器使用从`NonlinearOptimizerParams`继承的标准优化参数。

## 使用注意事项

- 当问题规模大且Hessian的计算不可行时，`NonlinearConjugateGradientOptimizer`最有效。
- 用户应根据其优化问题的具体特性选择合适的线搜索方法和共轭梯度更新公式。
- 在优化过程中监控误差和值可以提供对收敛行为的洞察，并帮助诊断潜在问题。

## 文件

- [NonlinearConjugateGradientOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearConjugateGradientOptimizer.h)
- [NonlinearConjugateGradientOptimizer.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearConjugateGradientOptimizer.cpp)

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearConjugateGradientOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## NonlinearEquality 非线性等式约束
# NonlinearEquality

GTSAM中的`NonlinearEquality`因子族提供了在变量之间或变量与常量值之间强制等式的约束。这些因子在需要严格等式约束的优化问题中很有用。以下是按功能分组的API概述。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearEquality.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## NonlinearEquality

`NonlinearEquality`因子强制变量与可行值之间的等式。它支持精确和非精确求值模式。

### 构造函数
- `NonlinearEquality(Key j, const T& feasible, const CompareFunction& compare)`
  创建一个强制键`j`处的变量与可行值`feasible`之间精确等式的因子。
  - `j`：要约束的变量的键。
  - `feasible`：要强制等式的可行值。
  - `compare`：可选的比较函数（默认使用`traits<T>::Equals`）。

- `NonlinearEquality(Key j, const T& feasible, double error_gain, const CompareFunction& compare)`
  创建一个允许带有指定误差增益的非精确求值的因子。
  - `error_gain`：当约束被违反时应用于误差的增益。

### 方法
- `double error(const Values& c) const`：计算给定值的误差。如果约束满足则返回`0.0`，如果启用了`allow_error_`则返回缩放后的误差。

- `Vector evaluateError(const T& xj, OptionalMatrixType H = nullptr) const`：为给定变量值`xj`求值误差向量。可选地计算Jacobian矩阵`H`。

- `GaussianFactor::shared_ptr linearize(const Values& x) const`：在给定值`x`处线性化因子以创建高斯因子。

## NonlinearEquality2

`NonlinearEquality2`因子是一个二值等式约束，强制两个变量之间的等式。

### 构造函数
- `NonlinearEquality2(Key key1, Key key2, double mu = 1e4)`
  创建一个强制`key1`和`key2`处变量之间等式的因子。
  - `key1`：第一个变量的键。
  - `key2`：第二个变量的键。
  - `mu`：约束强度（默认：`1e4`）。

### 方法
- `Vector evaluateError(const T& x1, const T& x2, OptionalMatrixType H1 = nullptr, OptionalMatrixType H2 = nullptr) const`：为给定变量值`x1`和`x2`求值误差向量。可选地计算Jacobian矩阵`H1`和`H2`。

- `void print(const std::string& s, const KeyFormatter& keyFormatter) const`：打印因子详细信息，包括键和噪声模型。

## 公共特性

### 误差处理模式
- 精确求值：如果线性化过程中违反了约束则抛出错误。
- 非精确求值：允许非零误差并使用`error_gain_`参数缩放它。

### 序列化
所有因子支持序列化以保存和加载模型。

### Testable接口
所有因子实现`Testable`接口，提供以下方法：
- `void print(const std::string& s, const KeyFormatter& keyFormatter) const`：打印因子详细信息。
- `bool equals(const NonlinearFactor& f, double tol) const`：检查两个因子在指定容差内是否相等。

这些因子为在非线性优化问题中强制等式约束提供了灵活的方式，使其对SLAM、机器人和控制系统等应用很有用。

## NonlinearFactor 非线性因子
# NonlinearFactor

## 概述

GTSAM中的`NonlinearFactor`类是非线性优化中使用的基本组件。它表示因子图中的一个因子。该类设计用于处理非线性连续函数。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 数学公式

`NonlinearFactor`通常由函数$f(x)$表示，其中$x$是变量向量。误差由下式给出：
$$
e(x) = f(x)- z
$$
其中$z$是观测测量值。优化过程旨在最小化平方误差之和：
$$
\min_x \sum_i \| e_i(x) \|^2
$$

线性化涉及在某点$x_0$周围近似$f(x)$：
$$
f(x) \approx f(x_0) + A\delta x
$$
其中$A$是$f$在$x_0$处的Jacobian矩阵，$\delta x \doteq x - x_0$。这导致线性化误差：
$$
e(x) \approx (f(x_0) + A\delta x) - z = A\delta x - b
$$
其中$b\doteq z - f(x_0)$是预测误差。

## 关键功能

### 误差计算

- **evaluateError**：此方法为给定的一组变量值计算因子的误差向量。误差通常是预测测量与实际测量之间的差。如果需要，该函数还可以返回Jacobian矩阵，这对于高斯-牛顿或列文伯格-马夸尔特等优化算法至关重要。

### Jacobian和Hessian

- **linearize**：此方法在线性化点周围线性化非线性因子。它返回一个`GaussianFactor`，这是使用一阶Taylor展开的`NonlinearFactor`的近似。这是迭代优化方法中的关键步骤，在该方法中问题被反复线性化和求解。

### 活动标志

- **active**：此函数检查因子是否应包含在优化过程中。如果因子不贡献误差，则可能不活动，这可能在条件约束或门控函数的情况下发生。

### 维度

- **dim**：返回因子的维度，对应于误差向量的大小。这对于理解因子对整体优化问题的贡献很重要。

### 键管理

- **keys**：提供对因子中涉及的键（或变量索引）的访问。这对于理解因子在因子图中连接到哪些变量至关重要。

## 使用注意事项

- `NonlinearFactor`类通常与`NonlinearFactorGraph`一起使用，后者是此类因子的集合。
- 用户需要在派生类中实现`evaluateError`方法来定义具体的测量模型。
- 该类设计灵活且可扩展，允许为特定应用创建自定义因子。

## NonlinearFactorGraph 非线性因子图
# NonlinearFactorGraph

## 概述

GTSAM中的`NonlinearFactorGraph`类是表示和求解非线性因子图的关键组件。因子图是一种二分图，表示函数的因子分解，通常用于概率图模型。在GTSAM的上下文中，它用于表示优化问题的结构，例如在同时定位与建图(SLAM)或运动推断结构(SfM)领域。

## 关键功能

### 构造和初始化

- **构造函数**：该类提供一个默认构造函数来初始化空的非线性因子图。

### 因子管理

- **add**：此方法允许向图中添加新因子。因子表示优化问题中的约束或测量。
- **reserve**：为指定数量的因子预分配空间，当因子数量事先已知时优化内存使用。

### 图操作

- **resize**：调整因子图的大小，这在动态修改图结构时可能很有用。
- **remove**：从图中移除由索引标识的因子。

### 查询和访问

- **size**：返回图中当前的因子数量。
- **empty**：检查图是否包含任何因子。
- **at**：通过索引访问特定因子。
- **back**：检索图中的最后一个因子。
- **front**：检索图中的第一个因子。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearFactorGraph.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

### 优化和线性化

- **linearize**：在给定线性化点处将非线性因子图转换为线性因子图。这是迭代优化算法如[Gauss-Newton](./GaussNewtonOptimizer.ipynb)或[Levenberg-Marquardt](./LevenbergMarquardtOptimizer.ipynb)中的关键步骤。

  线性化过程涉及计算非线性函数的Jacobian矩阵，得到线性近似：

  $$ f(x) \approx f(x_0) + A(x - x_0) $$

  其中$A$是在点$x_0$处求值的Jacobian矩阵。

## NonlinearISAM
# NonlinearISAM

## 概述

`NonlinearISAM`类封装了iSAM的增量因子分解版本{cite:p}`http://dx.doi.org/10.1109/TRO.2008.2006706`。它很大程度上被[iSAM2](./ISAM2.ipynb)取代，但它是一个更简单的类，功能较少，可能更容易调试。背景知识另请参见{cite:t}`https://doi.org/10.1561/2300000043`的最新小册子。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearISAM.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## NonlinearOptimizer 非线性优化器
# NonlinearOptimizer

## 概述

GTSAM中的`NonlinearOptimizer`类是（批量）非线性优化求解器的基类。它提供了优化非线性因子图的基本API，广泛用于机器人和计算机视觉应用中。

`NonlinearOptimizer`的主要目的是迭代地细化解的初始估计以最小化非线性代价函数。具体的优化算法如高斯-牛顿、列文伯格-马夸尔特和Dogleg在派生类中实现。

## 数学基础

`NonlinearOptimizer`中的优化过程基于迭代方法，求解非线性代价函数的最小值。通用方法涉及在当前估计处线性化非线性问题并求解生成的线性系统以更新估计。此过程重复直到满足收敛条件。

优化问题可以形式化定义为：

$$
\min_{x} \sum_{i} \| \phi_i(x) \|^2
$$

其中$x$是要优化的变量向量，$\phi_i(x)$是图中因子的残差。

## 关键方法

- `optimize()`方法是`NonlinearOptimizer`类的核心函数。它执行优化过程，迭代地更新估计以收敛到代价函数的局部最小值。
- `error()`方法计算当前估计的总误差。这通常是图中所有因子的平方误差之和。数学上，误差可以表达为：
    $$
    E(x) = \sum_{i} \| \phi_i(x) \|^2
    $$
    其中$\phi_i(x)$表示第$i$个因子的残差误差。
- `values()`方法返回当前的变量估计集合。这些估计在优化过程中更新。
- `iterations()`方法提供优化过程中执行的迭代次数。这对于分析优化器的收敛行为很有用。
- `params()`方法返回优化器使用的参数。这些参数可以包括收敛阈值、最大迭代次数和其他算法特定选项。

## 用法

`NonlinearOptimizer`类通常不直接使用。而是使用其派生类之一，如`GaussNewtonOptimizer`、`LevenbergMarquardtOptimizer`或`DoglegOptimizer`来执行特定类型的优化。这些派生类根据各自算法实现`optimize()`方法。

## 参数和配置

优化器可以使用`NonlinearOptimizerParams`进行配置和检测。从`NonlinearOptimizer`派生的类可以选择定义自己的参数，继承自`NonlinearOptimizerParams`，特定于其优化器实现。
可配置参数包括：
- `maxIterations` -- 优化器终止前的最大迭代次数（默认100）
- `relativeErrorTol` -- 停止迭代的最大相对误差下降（默认1e-5）
- `absoluteErrorTol` -- 停止迭代的最大绝对误差下降（默认1e-5）
- `errorTol` -- 停止迭代的最大总误差（默认0.0）
- `verbosity` -- 优化期间的打印详细程度（默认SILENT）
- `orderingType` -- 变量消元时使用的排序方法（默认COLAMD）
- `linearSolverType` -- 非线性优化器中使用的线性求解器类型（默认MULTIFRONTAL_CHOLESKY）
- `iterativeParams` -- 迭代优化参数的容器，用于CG求解器。

### IterationHook迭代钩子
`NonlinearOptimizerParams`还包含一个可选的`iterationHook`字段，如果设置，优化器在每次迭代后调用，允许对最近完成的迭代进行自定义行为响应。
iterationHook中提供的参数有：
 - `size_t iteration` -- 迭代计数
 - `double errorBefore` -- 迭代开始前的误差
 - `double errorAfter` -- 迭代完成后的误差

## 文件

- [NonlinearOptimizer.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearOptimizer.h)
- [NonlinearOptimizer.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearOptimizer.cpp)
- [NonlinearOptimizerParams.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearOptimizerParams.h)
- [NonlinearOptimizerParams.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/NonlinearOptimizerParams.cpp)

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/NonlinearOptimizer.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## PriorFactor 先验因子
# PriorFactor

## 概述

`PriorFactor`表示关于变量的高斯分布形式的先验信念。此类对于将先验知识纳入优化过程至关重要，可以显著增强解的准确性和鲁棒性。

## 关键功能

### PriorFactor构造

`PriorFactor`通过指定键、先验值和噪声模型来构造。键标识因子图中的变量，先验值表示变量的期望值，噪声模型封装与此先验信念相关的不确定性。

### 误差计算

`PriorFactor`的主要作用是计算变量的估计值与其先验之间的误差。您可以将误差视为：

$$
e(x) = x \ominus \mu
$$

其中$x$是估计值，$\mu$是先验均值，$\ominus$类似于减法。误差然后由噪声模型加权，形成此因子对总体目标函数的贡献。

### 添加到因子图

[NonlinearFactorGraph](./NonlinearFactorGraph.ipynb)有一个模板方法`addPrior<T>`，提供添加先验的便捷方式。

## 使用注意事项

- **噪声模型**：噪声模型的选择至关重要，因为它决定了先验被强制的力度。更紧的噪声模型意味着对先验更强的信念。*注意：非常强的先验可能会使待求解的线性系统的条件数非常高。在这种情况下，考虑添加[NonlinearEqualityFactor](./NonlinearEquality.ipynb)。*
- **与其他因子集成**：`PriorFactor`通常与对系统动态和测量建模的其他因子结合使用。它有助于锚定解，特别是在测量有限或噪声大的场景中。
- **应用**：常见应用包括SLAM（同时定位与建图），其中初始位姿或路标上的先验可以显著提高地图精度和收敛速度。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/PriorFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 示例笔记本

- [EKF_SLAM.ipynb](../../../python/gtsam/examples/EKF_SLAM.ipynb)
- [PlanarSlamExample.ipynb](../../../python/gtsam/examples/PlanarSLAMExample.ipynb)
- [RangeISAMExample_plaza2.ipynb](../../../python/gtsam/examples/RangeISAMExample_plaza2.ipynb)

## 备注

`PriorFactor`类派生自[ExtendedPriorFactor](ExtendedPriorFactor.ipynb)。

对于向量空间，我们有
$$
x \ominus \mu = x - \mu
$$
但误差*实际*上定义为
$$
- x.\text{localCoordinates}(\mu)
$$
其中`localCoordinates`是`retract`的逆。我们这样实现，是因为在$x$处的Jacobian是单位矩阵，这在计算上是有利的。

对于李群$\cal G$，其中$\text{localCoordinates}$实现为$\text{Log}$（指数映射$\text{Exp}$的逆），我们有
$$
- x.\text{localCoordinates}(\mu) = \text{Log}(\mu^{-1} x)
$$
然而，对于一般流形$\cal M$，可能不成立
$$
- x.\text{localCoordinates}(\mu) = \mu.\text{localCoordinates}(x)
$$
这实际上是有问题的（一个移动的目标）。但是，我们仍然选择以这种方式实现先验，否则"流形架构师"会被迫实现Jacobian。

## WhiteNoiseFactor 白噪声因子
# WhiteNoiseFactor

*以下是部分使用ChatGPT 4o生成的，需要验证。*

## 概述

GTSAM中的`WhiteNoiseFactor`是一个二值非线性因子，设计用于估计零均值高斯白噪声的参数。它使用**均值-精度参数化(mean-precision parameterization)**，其中均值$\mu$和精度$\tau = 1/\sigma^2$被作为要优化的变量处理。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/nonlinear/doc/WhiteNoiseFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass
```

## 参数化

该因子以如下方式对零均值高斯分布的负对数似然建模：
$$
f(z, \mu, \tau) = \log(\sqrt{2\pi}) - 0.5 \log(\tau) + 0.5 \tau (z - \mu)^2
$$
其中：
- $z$：测量值（观测数据）。
- $\mu$：高斯分布的均值（待估计）。
- $\tau$：高斯分布的精度$\tau = 1/\sigma^2$（也待估计）。

这种公式化允许因子同时优化噪声模型的均值和精度。

## 用例：估计IMU噪声特性

`WhiteNoiseFactor`可用于系统辨识任务，例如估计IMU的噪声特性。以下是其工作方式：

1. **定义测量**：
   - 在已知条件下（例如静止或匀速）收集一系列IMU测量（例如加速度计或陀螺仪读数）。

2. **设置因子图**：
   - 为每个测量向因子图添加`WhiteNoiseFactor`实例，将观测值$z$链接到均值和精度变量。

3. **优化图**：
   - 使用GTSAM的非线性优化工具求解最佳解释观测测量的均值$\mu$和精度$\tau$。

4. **提取噪声特性**：
   - 优化后的均值$\mu$表示传感器测量中的偏差。
   - 优化后的精度$\tau$可以取逆来计算标准差$\sigma = 1/\sqrt{\tau}$，它表示噪声水平。

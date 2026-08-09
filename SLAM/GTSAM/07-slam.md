# SLAM

`slam`模块提供了一系列常用于同时定位与建图(SLAM, Simultaneous Localization and Mapping)和运动推断结构(SfM, Structure from Motion)应用的因子、约束、工具和初始化算法。它构建在GTSAM核心推断引擎(`gtsam/inference`)和几何类型(`gtsam/geometry`)之上。

## 核心因子

这些是SLAM中常用作构建块的基础因子类型。
-   [PriorFactor](doc/PriorFactor.ipynb) : 仅作用于位姿变量旋转分量的先验因子。
-   [BetweenFactor](doc/BetweenFactor.ipynb) : 表示两个位姿或其他李群变量之间的相对测量（例如，来自[里程计(odometry)](https://en.wikipedia.org/wiki/Odometry)）。

## 视觉SLAM/SfM因子

专门为视觉数据（相机测量）设计的因子。

-   [GenericProjectionFactor](doc/GenericProjectionFactor.ipynb) : 标准单目投影因子，将3D路标、相机位姿和固定标定与2D测量关联起来。
-   [GeneralSFMFactor](doc/GeneralSFMFactor.ipynb) : 当相机标定未知或与位姿和路标一起优化时使用的投影因子。
-   [StereoFactor](doc/StereoFactor.ipynb) : 标准双目投影因子，将3D路标、相机位姿和固定双目标定与`StereoPoint2`测量关联起来。
-   [EssentialMatrixFactor](doc/EssentialMatrixFactor.ipynb) : 基于标定相机推导的本质矩阵(Essential matrix)来约束位姿或标定的因子。
-   [EssentialMatrixConstraint](doc/EssentialMatrixConstraint.ipynb) : 基于测量的本质矩阵约束两个相机之间相对位姿的因子。
-   [TriangulationFactor](doc/TriangulationFactor.ipynb) : 基于单个已知相机视角的测量来约束3D点的因子，用于三角化(triangulation)。
-   [PlanarProjectionFactor](doc/PlanarProjectionFactor.ipynb) : 专门为在2D平面上移动的机器人设计的投影因子。

## 智能因子(Smart Factors)

隐式管理路标变量、在优化过程中边缘化(marginalizing)它们的因子。

-   [SmartFactorParams](doc/SmartFactorParams.ipynb) : 控制智能因子行为的配置参数（线性化、退化处理等）。
-   [SmartProjectionFactor](doc/SmartProjectionFactor.ipynb) : 用于单目测量的智能因子，相机位姿和标定同时优化。
-   [SmartProjectionPoseFactor](doc/SmartProjectionPoseFactor.ipynb) : 用于单目测量的智能因子，相机标定固定，仅优化位姿。
-   [SmartProjectionRigFactor](doc/SmartProjectionRigFactor.ipynb) : 用于标定多相机系统的智能因子，仅优化系统的刚体位姿。
-   [SmartFactorBase](https://github.com/borglab/gtsam/blob/develop/gtsam/slam/SmartFactorBase.h) : 智能因子的抽象基类（内部使用）。

## 其他几何因子和约束

表示各种几何关系或约束的因子。

-   [PoseRotationPrior](doc/PoseRotationPrior.ipynb) : 仅作用于位姿变量旋转分量的先验因子。
-   [PoseTranslationPrior](doc/PoseTranslationPrior.ipynb) : 仅作用于位姿变量平移分量的先验因子。
-   [OrientedPlane3Factor](doc/OrientedPlane3Factor.ipynb) : 用于估计和约束3D平面路标(`OrientedPlane3`)的因子。
-   [RotateFactor](doc/RotateFactor.ipynb) : 基于未知旋转如何变换测量的旋转或方向来约束该旋转的因子。
-   [KarcherMeanFactor](doc/KarcherMeanFactor.ipynb) : 用于约束一组旋转或其他流形值的Karcher均值（几何平均）的因子。
-   [FrobeniusFactor](doc/FrobeniusFactor.ipynb) : 使用Frobenius范数直接作用于旋转矩阵元素的因子，是基于李代数因子的替代方案。
-   [ReferenceFrameFactor](doc/ReferenceFrameFactor.ipynb) : 将通过未知变换在两个不同坐标系中观察到的同一路标关联起来的因子，用于地图合并(map merging)。
-   [BoundingConstraint](doc/BoundingConstraint.ipynb) : 用于创建不等式约束的抽象基类（例如，保持变量在特定边界内）。需要C++派生。
-   [AntiFactor](doc/AntiFactor.ipynb) : 设计用于抵消另一个因子效果的因子，对于动态移除约束很有用。

## 初始化和工具

SLAM任务的辅助函数和类。

-   [lago](doc/lago.ipynb) : 用于初始化`Pose2`图的线性近似图优化(LAGO, Linear Approximation for Graph Optimization)。
-   [InitializePose3](doc/InitializePose3.ipynb) : 通过先求解旋转再求解平移来初始化`Pose3`图的方法。
-   [dataset](doc/dataset.ipynb) : 用于加载/保存常见SLAM数据集格式（g2o, TORO）的工具函数。
-   [expressions](https://github.com/borglab/gtsam/blob/develop/gtsam/slam/expressions.h) : 为常见SLAM因子类型预定义的Expression树（用于基于表达式的因子，内部使用）。

## CUDA加速

用于光束法平差(bundle adjustment)的GPU加速优化（需要CUDA构建；参见`gtsam/slam/cuda/`）。

-   [CudaSfmLevenbergMarquardtOptimizer](doc/CudaSfmLevenbergMarquardtOptimizer.ipynb) : 完全GPU驻留的列文伯格-马夸尔特(Levenberg-Marquardt)方法，用于BAL风格光束法平差（`GeneralSFMFactor<PinholeCamera<Cal3Bundler>, Point3>`图）；dense-Schur或cuDSS线性求解器。
-   [GNC with the CUDA SFM optimizer](doc/CudaSfmGncOptimizer.ipynb) : 使用`GncOptimizer`作为外层求解器、CUDA优化器作为内层求解器的鲁棒（离群值拒绝）光束法平差。

## SmartFactors 智能因子
# Smart Factors

GTSAM中的智能因子(smart factor)提供了一种高效处理涉及路标（如3D点）的约束的方式，用于SfM或SLAM问题中，而无需在优化状态中显式包含路标变量。相反，路标被隐式表示并被边缘化(marginalized out)，从而得到仅直接约束相机相关变量（位姿和/或标定）的因子。这通常会产生更小的状态空间，并能显著加速优化，特别是当使用迭代线性求解器时。

核心思想基于**Schur补(Schur complement)**。如果我们考虑一个涉及相机$C_i$和单个路标$p$的因子图，线性化系统的Hessian矩阵具有如下分块结构：

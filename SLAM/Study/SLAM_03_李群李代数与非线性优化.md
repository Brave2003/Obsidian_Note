# 李群李代数与非线性优化

## 学习资源推荐

### 核心资源

| 资源 | 类型 | 说明 |
|------|------|------|
| [SLAM十四讲 第4讲(李群) + 第6讲(非线性优化)](https://github.com/gaoxiang12/slambook2) | 书籍 | 中文最佳入门 |
| [A micro Lie theory for state estimation in robotics](https://arxiv.org/abs/1812.01537) | 论文 | Sola 2018, 李群实操圣经, 必读 |
| [State Estimation for Robotics - T. Barfoot](http://asrl.utias.utoronto.ca/~tdb/bib/barfoot_ser17.pdf) | 书籍 | 第7章Lie Groups, 数学推导最完整 |
| [Ceres Solver 教程](http://ceres-solver.org/tutorial.html) | 官方文档 | 非线性优化的工业标准, 例子很全 |
| [GTSAM 文档](https://gtsam.org/tutorials/) | 官方文档 | 因子图优化框架, 有理论说明 |
| [g2o 论文](https://ieeexplore.ieee.org/document/5970118) | 论文 | 图优化框架g2o的原始论文 |

### 补充阅读

| 资源 | 类型 | 说明 |
|------|------|------|
| [Sophus 库](https://github.com/strasdat/Sophus) | 代码库 | C++ Lie群实现, 配合SLAM十四讲 |
| [Manifold 库](https://github.com/artivis/manif) | 代码库 | 更现代化的C++ Lie群实现 |
| [Least Squares Optimization - Ceres](http://ceres-solver.org/nnls_tutorial.html) | 文档 | 最小二乘问题的完整讲解 |

---

## 核心概念详解

### 1. 为什么需要李群李代数？

**问题**：旋转矩阵 $R$ 满足 $R^T R = I$ 且 $\det(R) = 1$，这个约束在优化时很难处理（优化变量不能随意更新）。

**解决**：用李代数在单位元处的切空间来表示旋转，没有约束，可以自由加减。

### 2. SO(3) 和 SE(3)

#### SO(3) — 旋转群

- **群元素**：$R \in \mathbb{R}^{3\times3}, R^TR = I, \det(R) = 1$
- **李代数 so(3)**：$\phi \in \mathbb{R}^3$ (旋转向量)
- **指数映射**：$R = \exp(\phi^\wedge)$

**Rodrigues公式**（实际计算时用这个）：
$$R = I + \frac{\sin\theta}{\theta} \phi^\wedge + \frac{1-\cos\theta}{\theta^2} (\phi^\wedge)^2$$

其中 $\theta = \|\phi\|$

#### SE(3) — 刚体变换群

- **群元素**：$T = \begin{bmatrix} R & t \\ 0 & 1 \end{bmatrix} \in \mathbb{R}^{4\times4}$
- **李代数 se(3)**：$\xi = \begin{bmatrix} \rho \\ \phi \end{bmatrix} \in \mathbb{R}^6$
- $\rho$ 是平移分量 (3维), $\phi$ 是旋转分量 (3维)

### 3. 扰动模型 (核心中的核心)

**为什么要用扰动？** 指数映射前的雅可比太复杂，实际中都用扰动模型。

#### 左扰动

对 $R$ 施加左扰动：$R' = \exp(\delta \phi^\wedge) R$

$$\frac{\partial (Rp)}{\partial \delta \phi} = -(Rp)^\wedge$$

#### 右扰动

对 $R$ 施加右扰动：$R' = R \exp(\delta \phi^\wedge)$

$$\frac{\partial (Rp)}{\partial \delta \phi} = -R p^\wedge$$

**实际应用中**：左扰动对应全局坐标系中的旋转，右扰动对应局部坐标系。GTSAM 用右扰动，Ceres 通常用角轴参数化。

### 4. 非线性优化

#### 高斯-牛顿法 (Gauss-Newton)

对最小二乘问题 $\min_x \frac{1}{2} \|f(x)\|^2$:

$$J^T J \Delta x = -J^T f(x)$$

优点：收敛快（二次收敛速度）
缺点：$J^T J$ 可能半正定，导致求解不稳定

#### Levenberg-Marquardt

$$(J^T J + \lambda I) \Delta x = -J^T f(x)$$

$\lambda$ 小时 → 近似高斯牛顿
$\lambda$ 大时 → 近似梯度下降

优点：更鲁棒，实际工程中几乎都用LM

#### 鲁棒核函数

减少外点对优化结果的影响：

**Huber Loss**：
$$\rho(e) = \begin{cases} e^2/2 & |e| \leq \delta \\ \delta(|e| - \delta/2) & |e| > \delta \end{cases}$$

**Cauchy Loss**：
$$\rho(e) = \frac{c^2}{2} \log(1 + (e/c)^2)$$

### 5. Bundle Adjustment (BA)

同时优化相机位姿和3D点位置：

$$\min_{T_i, X_j} \sum_{i,j} \|u_{ij} - \pi(T_i, X_j)\|^2$$

**稀疏性**：每个误差项只涉及一个相机和一个点 → $H = J^T J$ 有稀疏结构 → Schur消元加速求解

### 6. 滑动窗口与边缘化

**为什么需要滑动窗口？**
- 全量BA随帧数增长，计算量无限增加
- 保留最近N帧（窗口内），丢弃旧帧

**直接用旧帧？** → 信息丢失
**正确做法：边缘化** → 将旧状态的信息转移到先验中

**边缘化的代价**：矩阵从稀疏变稠密（fill-in），但还是比全量优化小得多

### 7. 因子图

节点 = 待优化变量（位姿、路标点）
边 = 因子 = 约束（视觉残差、IMU残差、回环约束）

因子图的威力在于**即插即用**的模块化设计——每种传感器对应一种因子，组合起来就是完整系统。

---

## 工具使用要点

### Eigen 核心API

```cpp
// 旋转表示
Eigen::Matrix3d R;                    // 旋转矩阵
Eigen::AngleAxisd aa(theta, axis);    // 角轴
Eigen::Quaterniond q;                 // 四元数
Eigen::Vector3d euler = R.eulerAngles(0,1,2); // 欧拉角

// 变换
Eigen::Isometry3d T = Eigen::Isometry3d::Identity();
T.rotate(R);
T.pretranslate(t);

// 常见操作
q.normalized();
R = q.toRotationMatrix();
v.cross(w);  // 叉积
```

### Ceres 使用模式

```cpp
// 1. 定义残差
struct MyCost {
    template<typename T>
    bool operator()(const T* const x, T* residual) const {
        residual[0] = x[0] * x[0] - 4.0;
        return true;
    }
};

// 2. 添加残差块
ceres::Problem problem;
problem.AddResidualBlock(
    new ceres::AutoDiffCostFunction<MyCost, 1, 1>(new MyCost),
    new ceres::HuberLoss(1.0),  // 鲁棒核
    x
);

// 3. 求解
ceres::Solve(options, &problem, &summary);
```

### 参数化 (LocalParameterization)

当优化变量在流形上(如四元数只有3个自由度)时，需要告诉Ceres如何更新：

```cpp
// 四元数的过参数化
problem.SetParameterization(q, new ceres::EigenQuaternionParameterization);
```

---

## 动手实践清单

- [ ] 用 Eigen 实现 SO(3) 的指数/对数映射
- [ ] 实现高斯牛顿法求解简单的最小二乘问题
- [ ] 用 Ceres 做一次 Bundle Adjustment (PnP或曲线拟合)
- [ ] 比较有无 Huber Loss 对包含外点的优化结果
- [ ] 画一个因子图，标注节点和边的类型
- [ ] 阅读 VINS-Fusion 中边缘化代码
- [ ] 用 Sophus 实现左扰动/右扰动求导，数值验证

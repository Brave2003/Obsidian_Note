# Hybrid 混合模块

`hybrid`模块提供了专为涉及**离散和连续变量**的概率图模型中的推断而设计的类和算法。它扩展了`inference`模块的核心概念来处理这些混合变量类型，从而能够对具有离散模式、决策或状态以及连续状态的系统进行建模和求解。

## 核心混合概念

-   [HybridValues](doc/HybridValues.ipynb): 同时持有离散(`DiscreteValues`)、连续(`VectorValues`)和非线性(`Values`)变量赋值的容器。
-   [HybridFactor](doc/HybridFactor.ipynb): 涉及离散和/或连续变量的因子的抽象基类。
-   [HybridConditional](doc/HybridConditional.ipynb): 用于混合消元产生的条件概率（高斯、离散、HybridGaussian）的类型擦除(type-erased)封装。

## 混合因子类型

-   [HybridNonlinearFactor](doc/HybridNonlinearFactor.ipynb): 表示一个因子，其中离散选择从不同的`NonlinearFactor`(NoiseModelFactor)组件中进行选择，可能带有相关的标量能量。
-   [HybridGaussianFactor](doc/HybridGaussianFactor.ipynb): 表示一个因子，其中离散选择从不同的高斯因子组件中进行选择，可能带有相关的标量能量。
-   [HybridGaussianProductFactor](doc/HybridGaussianProductFactor.ipynb): 内部决策树结构，持有`GaussianFactorGraph`和`double`标量对，在混合消元期间使用。

## 混合因子图

-   [HybridFactorGraph](doc/HybridFactorGraph.ipynb): 设计用于持有混合因子的因子图基类。
-   [HybridNonlinearFactorGraph](doc/HybridNonlinearFactorGraph.ipynb): 包含非线性、离散和/或`HybridNonlinearFactor`类型的因子图，用于建模非线性混合系统。
-   [HybridGaussianFactorGraph](doc/HybridGaussianFactorGraph.ipynb): 包含高斯、离散和/或`HybridGaussianFactor`类型的因子图，通常由线性化得到。支持混合消元。

## 混合贝叶斯网和贝叶斯树

-   [HybridGaussianConditional](doc/HybridGaussianConditional.ipynb): 表示条件概率$P(X | M, Z)$，其中连续变量$X$依赖于离散父变量$M$和连续父变量$Z$，实现为`GaussianConditional`的决策树。
-   [HybridBayesNet](doc/HybridBayesNet.ipynb): 将`HybridGaussianFactorGraph`上的顺序变量消元结果表示为`HybridConditional`的DAG。
-   [HybridBayesTree](doc/HybridBayesTree.ipynb): 将`HybridGaussianFactorGraph`上的多前沿变量消元结果表示为团树，每个团包含一个`HybridConditional`。

## 增量混合推断

-   [HybridGaussianISAM](doc/HybridGaussianISAM.ipynb): 用于`HybridGaussianFactorGraph`的增量平滑与建图(ISAM)算法，基于更新`HybridBayesTree`。
-   [HybridNonlinearISAM](doc/HybridNonlinearISAM.ipynb): 为非线性混合问题提供ISAM接口的封装，管理线性化和底层的`HybridGaussianISAM`。
-   [HybridSmoother](doc/HybridSmoother.ipynb): 用于混合系统的增量固定滞后平滑器接口，管理对`HybridBayesNet`后验的更新。

## 混合消元中间结构

-   [HybridEliminationTree](doc/HybridEliminationTree.ipynb): 表示`HybridGaussianFactorGraph`顺序消元期间依赖关系的树结构。
-   [HybridJunctionTree](doc/HybridJunctionTree.ipynb): 多前沿消元中使用的中间簇树结构，在消元前将原始因子保存在团中。

## HybridBayesNet
# HybridBayesNet

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/hybrid/doc/HybridBayesNet.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not running on Colab, do nothing
```

`HybridBayesNet`表示专门为混合系统设计的有向图模型（贝叶斯网）。它是`gtsam.HybridConditional`对象的有序集合，按消元序列排列。

它扩展了`gtsam.BayesNet<HybridConditional>`，并允许将连续变量$X$和离散变量$M$上的联合概率分布$P(X, M)$表示为条件概率的乘积：
$$
P(X, M) = \prod_i P(\text{Frontal}_i | \text{Parents}_i)
$$
其中每个条件概率$P(\text{Frontal}_i | \text{Parents}_i)$存储为`HybridConditional`。此结构允许表示复杂的依赖关系，例如以离散模式为条件的连续变量($P(X|M)$)，以及纯离散($P(M)$)或纯连续($P(X)$)关系。

`HybridBayesNet`对象通常通过消元`HybridGaussianFactorGraph`获得。

```python
import gtsam
import numpy as np
import graphviz

from gtsam import (
    HybridConditional,
    GaussianConditional,
    DiscreteConditional,
    HybridGaussianConditional,
    HybridGaussianFactorGraph,
    HybridGaussianFactor,
    JacobianFactor,
    DecisionTreeFactor,
    Ordering,
)
from gtsam.symbol_shorthand import X, D
```

## 创建HybridBayesNet

虽然可以通过添加`HybridConditional`手动构建，但更常见的是通过消元`HybridGaussianFactorGraph`获得。

```python
# --- Method 1: Manual Construction ---
hbn_manual = gtsam.HybridBayesNet()

# P(D0)
dk0 = (D(0), 2)
cond_d0 = DiscreteConditional(dk0, [], "7/3") # P(D0=0)=0.7
hbn_manual.push_back(HybridConditional(cond_d0))

# P(X0 | D0)
dk0_parent = (D(0), 2)
 # Mode 0: P(X0 | D0=0) = N(0, 1)
gc0 = GaussianConditional(X(0), np.zeros(1), np.eye(1), gtsam.noiseModel.Unit.Create(1))
 # Mode 1: P(X0 | D0=1) = N(5, 4)
gc1 = GaussianConditional(X(0), np.array([2.5]), np.eye(1)*0.5, gtsam.noiseModel.Isotropic.Sigma(1,2.0))
cond_x0_d0 = HybridGaussianConditional(dk0_parent, [gc0, gc1])
hbn_manual.push_back(HybridConditional(cond_x0_d0))

# P(X1 | X0)
cond_x1_x0 = GaussianConditional(X(1), np.array([0.0]), np.eye(1), # d, R=I
                             X(0), np.eye(1),                  # Parent X0, S=I
                             gtsam.noiseModel.Isotropic.Sigma(1, 1.0)) # N(X1; X0, I)
hbn_manual.push_back(HybridConditional(cond_x1_x0))

print("Manually Constructed HybridBayesNet:")
# hbn_manual.print()
graphviz.Source(hbn_manual.dot())
```

```python
# --- Method 2: From Elimination ---
hgfg = HybridGaussianFactorGraph()
# P(D0) = 70/30
hgfg.push_back(DecisionTreeFactor([dk0], "0.7 0.3"))
# P(X0|D0) = mixture N(0,1); N(5,4)
# Factor version: 0.5*|X0-0|^2/1 + C0 ; 0.5*|X0-5|^2/4 + C1
factor_gf0 = JacobianFactor(X(0), np.eye(1), np.zeros(1), gtsam.noiseModel.Isotropic.Sigma(1, 1.0))
factor_gf1 = JacobianFactor(X(0), np.eye(1), np.array([5.0]), gtsam.noiseModel.Isotropic.Sigma(1, 2.0))
# Store -log(prior) for D0 in the hybrid factor (optional, could keep separate)
logP_D0_0 = -np.log(0.7)
logP_D0_1 = -np.log(0.3)
hgfg.push_back(HybridGaussianFactor(dk0, [(factor_gf0, logP_D0_0), (factor_gf1, logP_D0_1)]))
# P(X1|X0) = N(X0, 1)
hgfg.push_back(JacobianFactor(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), gtsam.noiseModel.Isotropic.Sigma(1, 1.0)))

print("\nOriginal HybridGaussianFactorGraph for Elimination:")
# hgfg.print()
graphviz.Source(hgfg.dot())
```

```python
# Note: Using HybridOrdering(hgfg) is generally recommended:
# it returns a Colamd constrained ordering where the discrete keys are
# eliminated after the continuous keys.
ordering = gtsam.HybridOrdering(hgfg)

hbn_elim, _ = hgfg.eliminatePartialSequential(ordering)
print("\nHybridBayesNet from Elimination:")
# hbn_elim.print()
graphviz.Source(hbn_elim.dot())
```

## HybridBayesNet上的操作

`HybridBayesNet`允许求值联合概率、采样、优化（找到MAP状态）以及提取边缘或条件分布。

```python
# Use the Bayes Net from elimination for consistency
hbn = hbn_elim

# --- Evaluation ---
values = gtsam.HybridValues()
values.insert(D(0), 0)
values.insert(X(0), np.array([0.1]))
values.insert(X(1), np.array([0.2]))

log_prob = hbn.logProbability(values)
prob = hbn.evaluate(values) # Same as exp(log_prob)
print(f"\nLogProbability P(X0=0.1, X1=0.2, D0=0): {log_prob}")
print(f"Probability P(X0=0.1, X1=0.2, D0=0): {prob}")

# --- Sampling ---
full_sample = hbn.sample()
print("\nSampled HybridValues:")
full_sample.print()

# --- Optimization (Finding MAP state) ---
# Computes MPE for discrete, then optimizes continuous given MPE
map_solution = hbn.optimize()
print("\nMAP Solution (Optimize):")
map_solution.print()

# --- MPE (Most Probable Explanation for Discrete Variables) ---
mpe_assignment = hbn.mpe()
print("\nMPE Discrete Assignment:")
print(mpe_assignment) # Should match discrete part of map_solution
```

```python
# --- Optimize Continuous given specific Discrete Assignment ---
dv = gtsam.DiscreteValues()
dv[D(0)] = 1
cont_solution_d0_eq_1 = hbn.optimize(dv)
print("\nOptimized Continuous Solution for D0=1:")
cont_solution_d0_eq_1.print()
```
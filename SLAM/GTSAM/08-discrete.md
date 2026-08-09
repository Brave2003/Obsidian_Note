# Discrete Inference in GTSAM GTSAM中的离散推断

GTSAM中的离散(discrete)模块为离散随机变量上的概率推断提供了全面的框架。该模块使用决策树(decision tree)和消元方法实现了贝叶斯网(Bayesian network)、因子图(factor graph)和高效的推断算法。

## 概述

GTSAM中的离散推断工作流程通常遵循以下模式：

1. **模型定义**：使用`DiscreteBayesNet`和`DiscreteConditional`从贝叶斯网开始
2. **似然集成**：通过`DiscreteFactorGraph`和各种因子类型转换为因子图
3. **推断**：消元变量以获得`DiscreteBayesTree`用于高效边缘化(marginalization)
4. **专门操作**：使用高级类进行特定推断任务

## 用于建模的贝叶斯网

### DiscreteBayesNet
用于表示离散变量上贝叶斯网的基础类。贝叶斯网将联合概率分布定义为条件概率分布的乘积。

```cpp
DiscreteBayesNet bayesNet;
bayesNet.add(conditional1);
bayesNet.add(conditional2);
```

**Notebook**: [DiscreteBayesNet.ipynb](doc/DiscreteBayesNet.ipynb)

### DiscreteConditional
表示贝叶斯网中的条件概率表P(X|Y1, Y2, ...)。这些构成贝叶斯网的构建块，编码变量之间的条件依赖关系。

```cpp
DiscreteConditional conditional(key, parents, signature);
```

**Notebook**: [DiscreteConditional.ipynb](doc/DiscreteConditional.ipynb)

### DiscreteDistribution
表示离散变量的边缘概率分布P(X)。用于先验。

**Notebook**: [DiscreteDistribution.ipynb](doc/DiscreteDistribution.ipynb)

### Signature
用于定义离散变量条件概率结构的便捷类，指定父-子关系和基数(cardinality)。

**Notebook**: [Signature.ipynb](doc/Signature.ipynb)

## 离散因子图

### DiscreteFactorGraph
因子图表示的核心类。贝叶斯网可以通过将条件概率转换为因子来转化为因子图，从而允许集成来自观测的似然因子。

```cpp
DiscreteFactorGraph graph = bayesNet.toFactorGraph();
graph.add(likelihoodFactor);
```

**Notebook**: [DiscreteFactorGraph.ipynb](doc/DiscreteFactorGraph.ipynb)

### DiscreteFactor
所有离散因子的抽象基类。因子表示离散变量上的任意函数，可以编码先验知识和似然信息。

### DecisionTreeFactor
使用决策树进行高效存储和计算的离散因子具体实现。对于具有条件独立性结构的因子尤其有效。

**Notebook**: [DecisionTreeFactor.ipynb](doc/DecisionTreeFactor.ipynb)

### TableFactor
将因子表示为显式的概率表。对于较小的因子或需要显式存储完全联合分布时很有用。

## 推断：贝叶斯树

### DiscreteBayesTree
对因子图进行变量消元的结果。贝叶斯树通过其团树(clique tree)结构实现高效的精确推断，支持快速的边缘化和条件化操作。

```cpp
DiscreteBayesTree bayesTree = graph.eliminateMultifrontal(ordering);
DiscreteValues result = bayesTree.optimize();
```

### DiscreteEliminationTree
消元过程中使用的中间结构。表示变量消元的计算树，在转换为贝叶斯树之前。

### DiscreteJunctionTree
用于推断的替代树结构，对于同一模型上的重复边缘查询特别有用。

## 专门和高级类

### DiscreteMarginals
提供因子图或贝叶斯树中所有变量的边缘概率高效计算。对于不确定性量化至关重要。

### DiscreteLookupDAG
用于离散概率模型中快速查找操作的特殊数据结构。针对重复查询场景进行了优化。

### DiscreteSearch
在离散概率空间中实现搜索算法，包括优化和采样方法。

### AlgebraicDecisionTree
实现具有代数操作（加法、乘法）的决策树，用于高效的因子操作。是许多离散运算的计算骨干。

### Assignment 和 DiscreteValues
用于表示离散模型中变量赋值和概率值的核心数据结构。

### DiscreteKey
表示具有其基数的离散随机变量，是定义变量空间的基础。

## 决策树和计算原语

### DecisionTree
支持任意值类型的基于模板的决策树实现。为高效离散计算提供基础。

### Ring
决策树值上代数操作的数学抽象，支持不同半环(semiring)上的泛型算法。

## 解析器和工具

### SignatureParser
解析条件概率签名的字符串表示，支持从文本方便地指定模型。

### TableDistribution
使用以表格表示的概率分布的工具类。

## 示例工作流程

### 基本贝叶斯网
1. 使用`DiscreteKey`定义变量
2. 使用`DiscreteConditional`创建条件概率
3. 使用`DiscreteBayesNet`构建网络
4. 查询边缘分布或采样

### 因子图推断
1. 从`DiscreteBayesNet`开始或直接构建`DiscreteFactorGraph`
2. 使用`DecisionTreeFactor`或`TableFactor`添加似然因子
3. 消元变量以获得`DiscreteBayesTree`
4. 使用`DiscreteMarginals`查询结果

### 高级操作
1. 使用`AlgebraicDecisionTree`进行自定义因子操作
2. 使用`DiscreteLookupDAG`进行重复查询
3. 应用`DiscreteSearch`解决优化问题

## DecisionTreeFactor
# DecisionTreeFactor

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/discrete/doc/DecisionTreeFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
try:
    import google.colab
    %pip install --quiet gtsam
except ImportError:
    pass  # Not running on Colab, do nothing
```

`DecisionTreeFactor`表示一组离散变量上的函数$f(X_1, X_2, ..., X_n)$。它是表示离散因子图中势函数或概率的通用构建块。

在内部，它使用`AlgebraicDecisionTree`(ADT)来存储函数值。这种表示可能非常高效，特别是对于许多赋值具有相同值（例如零）的稀疏因子。

`DecisionTreeFactor`是更专门因子（如`DiscreteConditional`）的基类。

```python
import gtsam
import numpy as np
import graphviz

from gtsam.symbol_shorthand import A, B
```

## 创建DecisionTreeFactor

因子由其`DiscreteKeys`（它依赖的变量）和一个值表定义。该表指定因子对其变量的每种可能赋值的输出。表中的值按最后一个键变化最快的顺序排列。

```python
# Define keys for two binary variables
KeyA = (A(0), 2)
KeyB = (B(0), 2)

# --- Method 1: From a spec string ---
# The values correspond to assignments (A,B) in the order:
# (0,0), (0,1), (1,0), (1,1)
f1_string = gtsam.DecisionTreeFactor([KeyA, KeyB], "1 2 3 4")
print("--- Factor f1 from string ---")
f1_string.print()

# --- Method 2: From a list of values ---
f2_vector = gtsam.DecisionTreeFactor([KeyA], [0.8, 0.2])
print("\n--- Factor f2 from vector ---")
f2_vector.print()
```

## DecisionTreeFactor上的操作

```python
# --- Evaluate ---
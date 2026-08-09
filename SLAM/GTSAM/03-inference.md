# Inference 推断模块

`inference`模块为GTSAM中的概率图模型(probabilistic graphical model)提供了基础类和算法，重点关注变量消元(variable elimination)及其生成的贝叶斯网(Bayes net)和贝叶斯树(Bayes tree)结构。

## 核心概念

-   [Key](doc/Key.ipynb)：用于唯一标识变量的基础类型(`uint64_t`)。
-   [Symbol](doc/Symbol.ipynb)：一种Key类型，编码一个字符和一个索引（例如`x0`）。
-   [LabeledSymbol](doc/LabeledSymbol.ipynb)：`Symbol`的变体，带有一个额外的标签字符，适用于多机器人场景（例如`xA0`）。
-   [EdgeKey](doc/EdgeKey.ipynb)：一种Key类型，编码一对32位整数。
-   [Factor](doc/Factor.ipynb)：所有因子（变量之间的关系）的抽象基类。
-   [FactorGraph](doc/FactorGraph.ipynb)：表示因子集合的基类。
-   [Conditional](doc/Conditional.ipynb)：从消元得到的条件分布/密度的抽象基类（$P(\text{Frontals} | \text{Parents})$）。

## 消元算法与控制

-   [Ordering](doc/Ordering.ipynb)：指定变量消元的顺序，对效率至关重要。
-   [VariableIndex](doc/VariableIndex.ipynb)：将变量映射到其所在的因子，用于高效的消元排序和构建。
-   [EliminateableFactorGraph](https://github.com/borglab/gtsam/blob/develop/gtsam/inference/EliminateableFactorGraph.h)：一个mixin类，为具体因子图类型（如`GaussianFactorGraph`、`SymbolicFactorGraph`）提供`eliminateSequential`和`eliminateMultifrontal`方法。

## 消元结果与结构

-   [BayesNet](doc/BayesNet.ipynb)：将顺序变量消元的结果表示为条件概率的有向无环图(DAG)。
-   [EliminationTree](doc/EliminationTree.ipynb)：表示顺序消元过程中依赖关系和计算的树结构。
-   [ClusterTree](doc/ClusterTree.ipynb)：节点为因子簇的树结构的基类（例如JunctionTree）。
-   [JunctionTree](doc/JunctionTree.ipynb)：一个簇树，表示多前沿消元(multifrontal elimination)过程中形成的团(clique)，在因子被消元成条件概率之前持有这些因子。
-   [BayesTreeCliqueBase](https://github.com/borglab/gtsam/blob/develop/gtsam/inference/BayesTreeCliqueBase.h)：BayesTree中节点（团）的抽象基类。
-   [BayesTree](doc/BayesTree.ipynb)：将多前沿变量消元的结果表示为团的树结构，每个团包含条件概率$P(\text{Frontals} | \text{Separator})$。

## 增量推断

-   [ISAM](doc/ISAM.ipynb)：基于更新BayesTree的增量平滑与建图(Incremental Smoothing and Mapping)算法（原始版本，通常在`nonlinear`模块中被ISAM2取代）。

## 可视化

-   [DotWriter](doc/DotWriter.ipynb)：辅助类，用于自定义Graphviz `.dot`文件的生成，以可视化图和树结构。

## BayesNet 贝叶斯网
# BayesNet

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

GTSAM中的`BayesNet`（贝叶斯网）表示一个有向图模型(directed graphical model)，通过对`FactorGraph`运行顺序变量消元（如Cholesky或QR分解）或从头构建来创建。

它本质上是一个`Conditional`对象的有序集合，按照消元顺序排列。每个条件概率表示$P(\text{variable} | \text{parents})$，其中父节点(parent)是在消元排序中出现较晚的变量。

贝叶斯网将联合概率分布表示为网中存储的条件概率的乘积：

$$
P(X_1, X_2, \dots, X_N) = \prod_{i=1}^N P(X_i | \text{Parents}(X_i))
$$
一个赋值的总对数概率是其各条件概率的对数概率之和：
$$
\log P(X_1, \dots, X_N) = \sum_{i=1}^N \log P(X_i | \text{Parents}(X_i))
$$

与`FactorGraph`类似，`BayesNet`以它存储的条件概率类型为模板参数（例如`GaussianBayesNet`、`DiscreteBayesNet`）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/BayesNet.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np
import graphviz

# We need concrete graph types and elimination to get a BayesNet
from gtsam import GaussianFactorGraph, Ordering, GaussianBayesNet
# For the Asia example
from gtsam import DiscreteBayesNet, DiscreteConditional, DiscreteKeys, DiscreteValues, symbol
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建BayesNet（通过消元）

BayesNet通常通过对`FactorGraph`进行消元获得。

```python
# Create a simple Gaussian Factor Graph P(x0) P(x1|x0) P(x2|x1)
graph = GaussianFactorGraph()
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
graph.add(X(0), -np.eye(1), np.zeros(1), model)
graph.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(X(1), -np.eye(1), X(2), np.eye(1), np.zeros(1), model)
print("Original Factor Graph:")
graph.print()
```

```python
# Eliminate sequentially using a specific ordering
ordering = Ordering([X(0), X(1), X(2)])
bayes_net = graph.eliminateSequential(ordering)

print("\nResulting BayesNet:")
bayes_net.print()
```

## 属性和访问

`BayesNet`提供对其组成部分和基本属性的访问。

```python
print(f"BayesNet size: {bayes_net.size()}")

# Access conditional by index
conditional1 = bayes_net.at(1)
print("Conditional at index 1: ")
conditional1.print()

# Get all keys involved
bn_keys = bayes_net.keys()
print(f"Keys in BayesNet: {bn_keys}")
```

## 求值和求解

`logProbability(Values)`方法根据贝叶斯网中的条件分布计算变量赋值的对数概率。对于高斯贝叶斯网，`optimize()`方法可用于通过回代(back-substitution)找到最大似然估计(MLE)解。

```python
# For GaussianBayesNet, we use VectorValues
mle_solution = bayes_net.optimize()

# Calculate log probability (requires providing values for all variables)
log_prob = bayes_net.logProbability(mle_solution)
print(f"Log Probability at {mle_solution.at(X(0))[0]:.0f},{mle_solution.at(X(1))[0]:.0f},{mle_solution.at(X(2))[0]:.0f}]: {log_prob}")

print("Optimized Solution (MLE):")
mle_solution.print()
```

## 可视化

贝叶斯网也可以使用Graphviz进行可视化。

```python
graphviz.Source(bayes_net.dot())
```

## 示例：DiscreteBayesNet（Asia网络）

虽然前面的示例集中在`GaussianBayesNet`上，但GTSAM也支持`DiscreteBayesNet`来表示离散变量的概率分布。这里我们通过直接添加`DiscreteConditional`对象来从零构建经典的"Asia"网络示例。

```python
# Define keys for the Asia network variables
A = symbol('A', 8) # Visit to Asia?
S = symbol('S', 7) # Smoker?
T = symbol('T', 6) # Tuberculosis?
L = symbol('L', 5) # Lung Cancer?
B = symbol('B', 4) # Bronchitis?
E = symbol('E', 3) # Tuberculosis or Lung Cancer?
X = symbol('X', 2) # Positive X-Ray?
D = symbol('D', 1) # Dyspnea (Shortness of breath)?

# Define cardinalities (all are binary in this case)
cardinalities = { A: 2, S: 2, T: 2, L: 2, B: 2, E: 2, X: 2, D: 2 }

# Helper to create DiscreteKeys object
def make_keys(keys_list):
    dk = DiscreteKeys()
    for k in keys_list:
        dk.push_back((k, cardinalities[k]))
    return dk
```

```python
# Create the DiscreteBayesNet
asia_net = DiscreteBayesNet()

# Helper function to create parent list in correct format
def make_parent_tuples(parent_keys):
    return [(pk, cardinalities[pk]) for pk in parent_keys]

# P(D | E, B) - Dyspnea given Either and Bronchitis
asia_net.add(DiscreteConditional((D, cardinalities[D]), make_parent_tuples([E, B]), "9/1 2/8 3/7 1/9"))

# P(X | E) - X-Ray result given Either
asia_net.add(DiscreteConditional((X, cardinalities[X]), make_parent_tuples([E]), "95/5 2/98"))

# P(E | T, L) - Either Tub. or Lung Cancer (OR gate)
# "F T T T" means P(E=1|T=0,L=0)=0, P(E=1|T=0,L=1)=1, P(E=1|T=1,L=0)=1, P(E=1|T=1,L=1)=1
asia_net.add(DiscreteConditional((E, cardinalities[E]), make_parent_tuples([T, L]), "F T T T"))

# P(B | S) - Bronchitis given Smoker
asia_net.add(DiscreteConditional((B, cardinalities[B]), make_parent_tuples([S]), "70/30 40/60"))

# P(L | S) - Lung Cancer given Smoker
asia_net.add(DiscreteConditional((L, cardinalities[L]), make_parent_tuples([S]), "99/1 90/10"))

# P(T | A) - Tuberculosis given Asia
asia_net.add(DiscreteConditional((T, cardinalities[T]), make_parent_tuples([A]), "99/1 95/5"))

# P(S) - Prior on Smoking
asia_net.add(DiscreteConditional((S, cardinalities[S]), [], "1/1")) # or "50/50"

# Add conditional probability tables (CPTs) using C++ sugar syntax
# P(A) - Prior on Asia
asia_net.add(DiscreteConditional((A, cardinalities[A]), [], "99/1"))

print("Asia Bayes Net:")
asia_net.print()
```

```python
# Visualize the network structure
dot_string = asia_net.dot()
display(graphviz.Source(dot_string))

# Evaluate the log probability of a specific assignment
# Example: Calculate P(A=0, S=0, T=0, L=0, B=0, E=0, X=0, D=0)
values = DiscreteValues()
for key, card in cardinalities.items():
    values[key] = 0 # Assign 0 to all variables to start

log_prob_zeros = asia_net.logProbability(values)
print(f"Log Probability of all zeros: {log_prob_zeros}")

# Sample from the Bayes Net
sample = asia_net.sample()
print("Sampled Values (basic print):")
print(sample)

# --- Pretty Print ---
print("Sampled Values (pretty print):")
# Create a reverse mapping from integer key to string like 'A8'
# We defined A=symbol('A',8), S=symbol('S',7), etc. above
symbol_map = { A: 'A8', S: 'S7', T: 'T6', L: 'L5', B: 'B4', E: 'E3', X: 'X2', D: 'D1' }
# Iterate through the sampled values and print nicely
# Sort items by the symbol string for consistent order (optional)
for key, value in sorted(sample.items(), key=lambda item: symbol_map.get(item[0], str(item[0]))):
    symbol_str = symbol_map.get(key, f"UnknownKey({key})") # Get 'A8' from key A
    print(f"  {symbol_str}: {value}")
```

## BayesTree 贝叶斯树
# BayesTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`BayesTree`（贝叶斯树）是一个图模型，表示在`FactorGraph`上进行多前沿变量消元的结果。它是一个树结构，其中每个节点是一个"团(clique)"，包含一组条件概率$P(\text{Frontals} | \text{Separator})$。

每个团k包含一个条件概率$P(F_k|S_k)$，其中$F_k$是前沿变量(frontal variable)，$S_k$是分隔符变量(separator variable)。贝叶斯树编码的联合概率分布由所有团条件概率的乘积给出：

$$
P(X) = \prod_k P(F_k | S_k)
$$

关键性质：
*   **团(Cliques)：** 每个节点（团）将一起消元的变量分组。
*   **前沿变量(Frontal Variables)：** 在特定团内被消元的变量。
*   **分隔符变量(Separator Variables)：** 一个团与其父团在树中共享的变量。这些变量在树的更高层被消元。
*   **树结构：** 比贝叶斯网更紧凑地表示消元过程中引入的依赖关系，尤其对于稀疏问题。

与`FactorGraph`和`BayesNet`类似，`BayesTree`以条件概率/团的类型为模板参数（例如`GaussianBayesTree`）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/BayesTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np
import graphviz

# We need concrete graph types and elimination to get a BayesTree
from gtsam import GaussianFactorGraph, Ordering, GaussianBayesTree, VariableIndex
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建BayesTree（通过消元）

BayesTree通常通过对`FactorGraph`执行多前沿消元来获得。

```python
# Create a simple Gaussian Factor Graph
graph = GaussianFactorGraph()
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
graph.add(X(0), -np.eye(1), np.zeros(1), model)           # Prior on x0
graph.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model) # x0 -> x1
graph.add(X(1), -np.eye(1), X(2), np.eye(1), np.zeros(1), model) # x1 -> x2
graph.add(L(1), -np.eye(1), X(0), np.eye(1), np.zeros(1), model) # l1 -> x0 (measurement)
graph.add(L(1), -np.eye(1), X(1), np.eye(1), np.zeros(1), model) # l1 -> x1 (measurement)
graph.add(L(2), -np.eye(1), X(1), np.eye(1), np.zeros(1), model) # l2 -> x1 (measurement)
graph.add(L(2), -np.eye(1), X(2), np.eye(1), np.zeros(1), model) # l2 -> x2 (measurement)

print("Original Factor Graph:")
display(graphviz.Source(graph.dot()))
```

```python
# Eliminate multifrontally using COLAMD ordering
ordering = Ordering.ColamdGaussianFactorGraph(graph)
# Note: Multifrontal typically yields multiple roots if graph is disconnected
bayes_tree = graph.eliminateMultifrontal(ordering)

print("Resulting BayesTree:")
bayes_tree.print()
print("\nVisualization:")
display(graphviz.Source(bayes_tree.dot()))
```

```python
print(f"BayesTree number of cliques: {bayes_tree.size()}")
```

## 求解和边缘分布

与`BayesNet`类似，`BayesTree`（特别是`GaussianBayesTree`等派生类型）提供`optimize()`方法来找到MLE解。它还允许使用信念传播(belief propagation)或树上的快捷求值(shortcut evaluation)高效计算单个变量上的边缘分布(marginal)或变量对上的联合边缘分布。

```python
# Optimize to find the MLE solution (for GaussianBayesTree)
mle_solution = bayes_tree.optimize()
print("Optimized Solution (MLE):")
mle_solution.print()

# Compute marginal factor on a single variable (returns a Conditional)
marginal_x1 = bayes_tree.marginalFactor(X(1))
print("\nMarginal Factor on x1:")
marginal_x1.print()

# Compute joint marginal factor graph on two variables
joint_x0_x2 = bayes_tree.joint(X(0), X(2))
print("\nJoint Marginal Factor Graph on (x0, x2):")
joint_x0_x2.print()
```

## ClusterTree 簇树
# ClusterTree

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/ClusterTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

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

`ClusterTree`（簇树）是比`EliminationTree`或`JunctionTree`更通用的结构。它表示一棵树，其中每个节点（一个"簇(Cluster)"）包含来自原始因子图的一部分因子。关键性质是该树必须是"族保持(family preserving)"的，意味着每个原始因子必须完全属于单个簇内。

`ClusterTree`本身是一个基类。`EliminatableClusterTree`添加了执行消元的能力，而`JunctionTree`是一种特定类型的`EliminatableClusterTree`，从`EliminationTree`派生而来。

在Python应用中直接使用`ClusterTree`不太常见；请参见其子类。

## Conditional 条件概率
# Conditional

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`gtsam.Conditional`是从变量消元得到的条件概率分布或密度的基类。

令$F$为前沿变量的集合，$S$为父（分隔符）变量的集合。一个条件概率表示：

$$
P(F | S)
$$
方法`evaluate`、`logProbability`和`error`之间有以下关系：
$$
\text{evaluate}(F, S) = P(F | S)
$$
$$
\text{logProbability}(F, S) = \log P(F | S)
$$
$$
\text{logProbability}(F, S) = -(\text{negLogConstant} + \text{error}(F, S))
$$
其中`negLogConstant`是确保$\int P(F|S) dF = 1$的归一化常数$k$的$-\log k$。

与`gtsam.Factor`类似，您通常不直接实例化`gtsam.Conditional`。相反，您使用从消元获得的派生类，例如：
*   `gtsam.GaussianConditional`
*   `gtsam.DiscreteConditional`
*   `gtsam.HybridGaussianConditional`
*   `gtsam.SymbolicConditional`

本笔记演示基类提供的通用接口。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Conditional.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np

# We need concrete graph types and elimination to get a Conditional
from gtsam import GaussianFactorGraph, Ordering
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 示例：获取和检查一个Conditional

我们将创建一个简单的`GaussianFactorGraph`并消元一个变量以获得一个`GaussianConditional`。

```python
# Create a simple Gaussian Factor Graph P(x0) P(x1|x0)
graph = GaussianFactorGraph()
model1 = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
model2 = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)

# Prior on x0
graph.add(X(0), -np.eye(1), np.zeros(1), model1)
# Factor between x0 and x1
graph.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model2)

print("Eliminating x0 from graph:")
graph.print()

# Eliminate x0
ordering = Ordering([X(0)])
bayes_net, remaining_graph = graph.eliminatePartialSequential(ordering)

print("\nResulting BayesNet:")
bayes_net.print()

# Get the resulting conditional P(x0 | x1)
# In this case, it's a GaussianConditional
conditional = bayes_net.at(0) # or bayes_net[0]

# Access methods from the Conditional base class
print(f"Conditional Keys (all): {conditional.keys()}")
print(f"First Frontal Key: {conditional.firstFrontalKey()} ({gtsam.DefaultKeyFormatter(conditional.firstFrontalKey())})")

# Conditional objects can also be printed
# conditional.print("P(x0 | x1): ")
```

## 求值（派生类方法）

具体的条件概率类提供诸如`logProbability(values)`或`evaluate(values)`的方法，以计算给定父变量值下的条件概率（或密度）。这些方法定义在派生类中，而非`Conditional`基类本身。

```python
# Example for GaussianConditional (requires VectorValues)
vector_values = gtsam.VectorValues()
vector_values.insert(X(0), np.array([0.0])) # Value for frontal variable
vector_values.insert(X(1), np.array([1.0])) # Value for parent variable

# These methods are specific to GaussianConditional / other concrete types
try:
    log_prob = conditional.logProbability(vector_values)
    print(f"\nLog Probability P(x0|x1=1.0): {log_prob}")
    prob = conditional.evaluate(vector_values)
    print(f"Probability P(x0|x1=1.0): {prob}")
except AttributeError:
    print("\nNote: logProbability/evaluate called on base Conditional pointer, needs derived type.")
    # In C++, you'd typically have a shared_ptr<GaussianConditional>.
    # In Python, if you know the type, you might access methods directly,
    # but the base class wrapper doesn't expose derived methods.
    pass

# To properly evaluate, you often use the BayesNet/BayesTree directly
bayes_net.logProbability(vector_values)
```

## DotWriter
# DotWriter

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`DotWriter`类是GTSAM中的一个辅助工具，用于自定义Graphviz `.dot`文件字符串的生成，以可视化因子图、贝叶斯网和贝叶斯树。

它允许您控制诸如图形大小、因子是否绘制为点、边的绘制方式以及为变量和因子指定显式位置等方面。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/DotWriter.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import DotWriter
from gtsam import SymbolicFactorGraph # Example graph type
from gtsam import symbol_shorthand
import graphviz # For rendering
import numpy as np

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建和配置DotWriter

您创建一个`DotWriter`对象，然后可以修改其公共成员变量来更改输出格式。

```python
writer = DotWriter(
    figureWidthInches = 8.0,
    figureHeightInches = 5.0,
    plotFactorPoints = True,      # Draw black dots for factors
    connectKeysToFactor = True,   # Draw edges from variables to factor dots
    binaryEdges = False           # Don't simplify binary factors to single edges
)

# --- Configuration Options ---

# Specify explicit positions (used by neato -n)
writer.variablePositions = {
    X(0): gtsam.Point2(0, 0),
    X(1): gtsam.Point2(2, 0),
    X(2): gtsam.Point2(4, 0),
    L(1): gtsam.Point2(1, 2),
    L(2): gtsam.Point2(3, 2)
}

# Specify position hints (alternative, uses symbol char and index)
# writer.positionHints = {'x': 0.0, 'l': 2.0} # Puts 'x' vars at y=0, 'l' vars at y=2

# Specify which variables should be boxes
writer.boxes = {L(1), L(2)}

# Specify factor positions (less common)
# writer.factorPositions = {3: gtsam.Point2(0.5, 1.0)}
```

## 与图对象一起使用

配置好的`DotWriter`对象作为参数传递给`FactorGraph`、`BayesNet`或`BayesTree`的`.dot()`方法。

```python
# Create the same graph as in VariableIndex example
graph = SymbolicFactorGraph()
graph.push_factor(X(0))           # Factor 0
graph.push_factor(X(0), X(1))     # Factor 1
graph.push_factor(X(1), X(2))     # Factor 2
graph.push_factor(X(0), L(1))     # Factor 3
graph.push_factor(X(1), L(1))     # Factor 4
graph.push_factor(X(1), L(2))     # Factor 5
graph.push_factor(X(2), L(2))     # Factor 6

# Generate dot string using the configured writer
dot_string = graph.dot(writer=writer)

# Render the graph
graphviz.Source(dot_string)
```

## EdgeKey
# EdgeKey

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`EdgeKey`是GTSAM中的一个工具类，用于将一对32位无符号整数编码为单个64位的`gtsam.Key`。这对表示图中的边或其他成对关系很有用，其中每个元素适合32位。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/EdgeKey.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import EdgeKey
```

## 初始化

可以通过提供两个32位无符号整数（`i`和`j`）创建`EdgeKey`。也可以通过解码现有的`gtsam.Key`（整数）来创建，假设它是使用`EdgeKey`格式编码的。

```python
# Create EdgeKey from integers i=10, j=20
ekey1 = EdgeKey(10, 20)
print(f"EdgeKey from (10, 20): {ekey1}") # Uses __str__ which calls operator std::string

# Get the underlying integer key
key1 = ekey1.key()

# Reconstruct EdgeKey from the key
ekey2 = EdgeKey(key1)
print(f"EdgeKey from key {key1}: {ekey2}")
```

## 属性和用法

您可以访问原始的`i`和`j`值以及组合后的`Key`。

```python
edge = EdgeKey(123, 456)

print(f"EdgeKey: {edge}")
print(f"  i: {edge.i()}")
print(f"  j: {edge.j()}")
print(f"  Key: {edge.key()}")
```

## EliminationTree 消元树
# EliminationTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`EliminationTree`（消元树）表示给定特定`Ordering`下在`FactorGraph`上进行顺序变量消元（如高斯消元）的计算结构。

树中的每个节点对应一个被消元的变量。节点的子节点表示更早被消元并产生了涉及父节点变量的因子的变量。涉及节点处变量的因子存储在该节点中。

消去一个`EliminationTree`得到`BayesNet`。

虽然对理论很重要，但在Python中直接操作`EliminationTree`对象不如在`FactorGraph`上使用`eliminateSequential`方法常见，后者内部使用`EliminationTree`。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/EliminationTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np

# EliminationTree is templated, need concrete types
from gtsam import GaussianFactorGraph, Ordering, GaussianEliminationTree, GaussianBayesNet
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建EliminationTree

`EliminationTree`从`FactorGraph`和`Ordering`构造。

```python
# Create a graph (same as BayesTree example)
graph = GaussianFactorGraph()
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
graph.add(X(0), -np.eye(1), np.zeros(1), model)
graph.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(X(1), -np.eye(1), X(2), np.eye(1), np.zeros(1), model)
graph.add(L(1), -np.eye(1), X(0), np.eye(1), np.zeros(1), model)
graph.add(L(1), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(L(2), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(L(2), -np.eye(1), X(2), np.eye(1), np.zeros(1), model)

# Define an ordering
ordering = Ordering([X(0), L(1), X(1), L(2), X(2)])

# Construct the Elimination Tree
elimination_tree = GaussianEliminationTree(graph, ordering)

elimination_tree.print("Elimination Tree: ")
```

## 消元

`EliminationTree`的主要用途是执行顺序消元以生成`BayesNet`。

```python
# The eliminate function needs to be specified (e.g., EliminateGaussian)
# In Python, this is usually handled internally by graph.eliminateSequential
# but the C++ EliminationTree has an eliminate method.

# Let's call the graph's eliminateSequential which uses the tree internally
bayes_net, remaining_graph = graph.eliminatePartialSequential(ordering)

print("BayesNet from EliminationTree:")
bayes_net.print()
```

## Factor 因子
# Factor

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`gtsam.Factor`是GTSAM中所有因子的抽象基类，包括非线性因子、高斯因子、离散因子和条件概率。它定义了所有因子共同的接口，主要围绕因子涉及的变量集（键）展开。

您通常不直接实例化`gtsam.Factor`，而是使用其派生类，如`gtsam.NonlinearFactor`、`gtsam.JacobianFactor`、`gtsam.DiscreteFactor`等。

一个因子的`error`函数通常通过负对数似然与概率或似然$P(X)$或$\phi(X)$关联：

$$
\text{error}(X) = - \log \phi(X) + K
$$
或等效地：
$$
\phi(X) \propto \exp(-\text{error}(X))
$$
其中$X$是因子涉及的变量，$\phi(X)$是势函数(potential function)（与概率或似然成正比），$K$是某个常数。最小化error对应于最大化因子表示的概率/似然。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Factor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam.utils.test_case import GtsamTestCase

# We need a concrete factor type for demonstration
from gtsam import PriorFactorPose2, BetweenFactorPose2, Pose2, Point3
from gtsam import symbol_shorthand

X = symbol_shorthand.X
```

## 基本接口

所有因子都提供访问其涉及变量的键的方法。

```python
noise_model = gtsam.noiseModel.Diagonal.Sigmas(Point3(0.1, 0.1, 0.05))

# Create some concrete factors
prior_factor = PriorFactorPose2(X(0), Pose2(0, 0, 0), noise_model)
between_factor = BetweenFactorPose2(X(0), X(1), Pose2(1, 0, 0), noise_model)

# Access keys (methods inherited from gtsam.Factor)
prior_keys = prior_factor.keys()
print(f"Prior factor keys: {prior_keys} ({gtsam.DefaultKeyFormatter(prior_keys[0])})")
print(f"Prior factor size: {prior_factor.size()}")

between_keys = between_factor.keys()
print(f"Between factor keys: {between_keys} ({gtsam.DefaultKeyFormatter(between_keys[0])}, {gtsam.DefaultKeyFormatter(between_keys[1])})")
print(f"Between factor size: {between_factor.size()}")

print(f"Is prior factor empty? {prior_factor.empty()}")

# Factors can be printed
prior_factor.print("Prior Factor: ")
```

## 误差函数

许多因子类型（尤其是非线性和高斯因子）的关键方法是`error(Values)`。它评估在给定特定变量赋值下因子的负对数似然。对于优化，目标通常是找到最小化总误差（所有因子误差之和）的`Values`。

```python
values = gtsam.Values()
values.insert(X(0), Pose2(0, 0, 0))
values.insert(X(1), Pose2(1, 0, 0))

# Evaluate error (example with BetweenFactor)
error1 = between_factor.error(values)
print(f"Error at ground truth: {error1}")

# Change a value and recalculate error
values.update(X(1), Pose2(0, 0, 0))
error2 = between_factor.error(values)
print(f"Error with incorrect x1: {error2:.1f}")
```

## FactorGraph 因子图
# FactorGraph

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`FactorGraph`表示一个因子图(factor graph)，即连接变量和因子的二分图。在GTSAM中，`FactorGraph`类（及其模板实例化如`GaussianFactorGraph`、`NonlinearFactorGraph`等）主要存储一个因子集合。

此类是不同类型因子图的基类。您通常使用具体的实例化，如`gtsam.GaussianFactorGraph`或`gtsam.NonlinearFactorGraph`。

因子图表示的总概率$P(X)$与各个因子势函数$\phi_i$的乘积成正比：
$$
P(X) \propto \prod_i \phi_i(X_i)
$$
其中$X_i$是因子$i$涉及的变量。用误差（负对数似然）表示：
$$
P(X) \propto \exp\left(-\sum_i \text{error}_i(X_i)\right)
$$
在给定赋值$X$下，图的总误差是个别因子误差之和：
$$
\text{error}(X) = \sum_i \text{error}_i(X_i)
$$

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/FactorGraph.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np
import graphviz

# Example uses NonlinearFactorGraph, but concepts apply to others
from gtsam import NonlinearFactorGraph, PriorFactorPose2, BetweenFactorPose2, Pose2, Point3
from gtsam import symbol_shorthand

X = symbol_shorthand.X
```

## 初始化和添加因子

通常创建空的`FactorGraph`，然后逐个或从容器中添加因子。

```python
graph = NonlinearFactorGraph()

# Define noise models
prior_noise = gtsam.noiseModel.Diagonal.Sigmas(Point3(0.1, 0.1, 0.05))
odometry_noise = gtsam.noiseModel.Diagonal.Sigmas(Point3(0.2, 0.2, 0.1))

# Create factors
factor1 = PriorFactorPose2(X(0), Pose2(0, 0, 0), prior_noise)
factor2 = BetweenFactorPose2(X(0), X(1), Pose2(1, 0, 0), odometry_noise)
factor3 = BetweenFactorPose2(X(1), X(2), Pose2(1, 0, 0), odometry_noise)

# Add factors to the graph
graph.add(factor1)  # add is synonym for push_back
graph.push_back(factor2)

print(f"Graph size after adding factors: {graph.size()}")
```

## 访问因子和属性

```python
print(f"Is graph empty? {graph.empty()}")
print(f"Number of factors (size): {graph.size()}")
print(f"Number of non-null factors (nrFactors): {graph.nrFactors()}") # Useful if factors were removed

# Access factor by index
retrieved_factor = graph.at(1)
print("Factor at index 1: ")
retrieved_factor.print()

# Get all unique keys involved in the graph
all_keys = graph.keys() # Returns a KeySet
print(f"Keys involved in the graph: {all_keys}")

# Iterate through factors
# for i, factor in enumerate(graph):
#     if factor:
#         print(f"Factor {i} keys: {factor.keys()}")
```

## 图误差

`error(Values)`方法计算在给定特定变量赋值下图的总体误差。这是每个个别因子误差之和。

```python
values = gtsam.Values()
values.insert(X(0), Pose2(0, 0, 0))
values.insert(X(1), Pose2(1, 0, 0))
values.insert(X(2), Pose2(2, 0, 0))

total_error1 = graph.error(values)
print(f"Total graph error at ground truth: {total_error1}")

# Introduce an error
values.update(X(2), Pose2(1, 0, 0))
total_error2 = graph.error(values)
print(f"Total graph error with incorrect x2: {total_error2:.1f}")
```

## 图可视化

因子图可以使用Graphviz通过`dot()`方法进行可视化。

```python
graphviz.Source(graph.dot(values))
```

## 消元

因子图的一个关键目的是通过变量消元进行推断。`FactorGraph`本身不执行消元，但其派生类（如`GaussianFactorGraph`、`SymbolicFactorGraph`）从`EliminateableFactorGraph`继承了`eliminateSequential`和`eliminateMultifrontal`方法。

## ISAM
# ISAM

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`gtsam.ISAM`（增量平滑与建图, Incremental Smoothing and Mapping）是一个继承自`BayesTree`并添加了`update`方法的类。该方法允许在向问题中添加新因子（例如新测量）时，对解进行高效的增量更新。

iSAM不是从头重新消元整个因子图，而是识别贝叶斯树中受新因子影响的部分，移除该部分，仅重新消元必要的变量，将结果合并回现有树中。

与`BayesTree`类似，它是模板化的（例如，`GaussianISAM`继承自`GaussianBayesTree`）。对于需要增量更新的实际应用，`ISAM2`通常更受青睐，因为它有进一步的优化，如流式重线性化(fluid relinearization)和变量移除支持，但`ISAM`展示了基于贝叶斯树的核心增量更新概念。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/ISAM.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np

# Use Gaussian variants for demonstration
from gtsam import GaussianFactorGraph, Ordering, GaussianISAM, GaussianBayesTree
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 初始化

`ISAM`对象可以空创建，也可以从现有的`BayesTree`初始化。

```python
# Create an empty ISAM object
isam1 = GaussianISAM()

# Create from an existing Bayes Tree (e.g., from an initial batch solve)
initial_graph = GaussianFactorGraph()
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
initial_graph.add(X(0), -np.eye(1), np.zeros(1), model) # Prior on x0

initial_bayes_tree = initial_graph.eliminateMultifrontal(Ordering([X(0)]))
print("Initial BayesTree:")
initial_bayes_tree.print()

isam2 = GaussianISAM(initial_bayes_tree)
print("ISAM from BayesTree:")
isam2.print()
```

## 增量更新

核心功能是`update(newFactors)`方法。

```python
# Start with the ISAM object containing the prior on x0
isam = GaussianISAM(initial_bayes_tree)
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)

# --- First Update ---
new_factors1 = GaussianFactorGraph()
new_factors1.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model) # x0 -> x1
isam.update(new_factors1)

print("ISAM after first update (x0, x1):")
isam.print()

# --- Second Update ---
new_factors2 = GaussianFactorGraph()
new_factors2.add(X(1), -np.eye(1), X(2), np.eye(1), np.zeros(1), model) # x1 -> x2
isam.update(new_factors2)

print("\nISAM after second update (x0, x1, x2):")
isam.print()
```

## 求解和边缘分布

由于`ISAM`继承自`BayesTree`，您可以在执行更新后使用相同的方法，如`optimize()`和`marginalFactor()`。

```python
# Get the solution from the final ISAM state
solution = isam.optimize()
print("Optimized Solution after updates:")
solution.print()
```

## JunctionTree 联结树
# JunctionTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`JunctionTree`（联结树）是GTSAM多前沿变量消元中使用的一个中间数据结构。它是一个`ClusterTree`，其中每个节点（簇）对应于消元过程中形成的弦图(chordal graph)中的一个团。

与相关结构的关键区别：
*   **与EliminationTree对比：** 联结树节点可以表示多个变量同时消元（一个"前沿"集），而消元树节点通常表示单个变量消元。
*   **与BayesTree对比：** JunctionTree节点包含与该团中消元的变量相关的原始因子。BayesTree节点包含消去这些因子的*结果*（即条件概率$P(\text{Frontals} | \text{Separator})$）。

与`EliminationTree`类似，在Python中直接操作`JunctionTree`对象不太常见。它主要是在生成`BayesTree`时`eliminateMultifrontal`使用的内部结构。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/JunctionTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
import numpy as np

# JunctionTree is templated, need concrete types
from gtsam import GaussianFactorGraph, Ordering, VariableIndex
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建JunctionTree

`JunctionTree`通常作为多前沿消元过程的一部分从`EliminationTree`构造。直接构造函数可能不会在Python中暴露，因为它通常在内部创建。

```python
# Create a graph (same as BayesTree example)
graph = GaussianFactorGraph()
model = gtsam.noiseModel.Isotropic.Sigma(1, 1.0)
graph.add(X(0), -np.eye(1), np.zeros(1), model)
graph.add(X(0), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(X(1), -np.eye(1), X(2), np.eye(1), np.zeros(1), model)
graph.add(L(1), -np.eye(1), X(0), np.eye(1), np.zeros(1), model)
graph.add(L(1), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(L(2), -np.eye(1), X(1), np.eye(1), np.zeros(1), model)
graph.add(L(2), -np.eye(1), X(2), np.eye(1), np.zeros(1), model)

ordering = Ordering.ColamdGaussianFactorGraph(graph)

# Perform multifrontal elimination, which uses a JunctionTree internally
bayes_tree, remaining_graph = graph.eliminatePartialMultifrontal(ordering)

# The resulting BayesTree reflects the structure of the intermediate JunctionTree
print("Resulting BayesTree (structure mirrors JunctionTree):")
bayes_tree.print()

# Accessing the JunctionTree directly isn't typical in Python workflows.
# Its structure is implicitly captured by the BayesTree cliques.
```

## Key
# Key

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

GTSAM中的`Key`只是`std::uint64_t`的`typedef`。它作为因子图中变量或`Values`容器中值的唯一标识符。虽然您可以使用原始整数键，但GTSAM提供了诸如`Symbol`和`LabeledSymbol`的辅助类，以创建在64位整数内编码类型和索引信息的语义上有意义的键。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Key.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import Symbol, LabeledSymbol
import numpy as np
```

## 基本用法

键通常使用`Symbol`或`LabeledSymbol`创建，然后隐式或显式转换为`Key`类型（整数）。

```python
sym = Symbol('x', 0)
key_from_symbol = sym.key() # Or just 'sym' where a Key is expected
print(f"Symbol Key (x0): {key_from_symbol}")
print(f"Type: {type(key_from_symbol)}")

lsym = LabeledSymbol(ord('a'), ord('B'), 1)
key_from_labeled_symbol = lsym.key()
print(f"LabeledSymbol Key (aB1): {key_from_labeled_symbol}")
print(f"Type: {type(key_from_labeled_symbol)}")

# You can also use plain integers, but it's less descriptive
plain_key = 12345
print(f"Plain Integer Key: {plain_key}")
print(f"Type: {type(plain_key)}")
```

## Key格式化

当打印包含键的GTSAM对象（如因子图或Values）时，您可以指定`KeyFormatter`来控制键的显示方式。默认格式化器尝试将键解释为`Symbol`。

```python
print("Default Formatter:")
print(f"  Symbol Key: {gtsam.DefaultKeyFormatter(key_from_symbol)}")
print(f"  LabeledSymbol Key: {gtsam.DefaultKeyFormatter(key_from_labeled_symbol)}")
print(f"  Plain Key: {gtsam.DefaultKeyFormatter(plain_key)}")

# Example of a custom formatter
def my_formatter(key):
    # Try interpreting as LabeledSymbol, then Symbol, then default
    try:
        lsym = gtsam.LabeledSymbol(key)
        if lsym.label() != 0: # Check if it's likely a valid LabeledSymbol
             return f"KEY[{lsym.string()}]"
    except:
        pass
    try:
        sym = gtsam.Symbol(key)
        if sym.chr() != 0: # Check if it's likely a valid Symbol
            return f"KEY[{sym.string()}]"
    except:
        pass
    return f"KEY[{key}]"

print("Custom Formatter:")
print(f"  Symbol Key: {my_formatter(key_from_symbol)}")
print(f"  LabeledSymbol Key: {my_formatter(key_from_labeled_symbol)}")
print(f"  Plain Key: {my_formatter(plain_key)}")

# KeyVectors, KeyLists, KeySets can also be printed using formatters
key_vector = gtsam.KeyVector([key_from_symbol, key_from_labeled_symbol, plain_key])
# key_vector.print("My Vector: ", my_formatter) # .print() method uses formatter directly
```

## LabeledSymbol
# LabeledSymbol

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`LabeledSymbol`是`gtsam.Symbol`的专门版本，主要设计用于多机器人应用或除类型字符和索引外还需要额外标签的场景。它将一个类型字符（`unsigned char`）、一个标签字符（`unsigned char`）和一个索引（`uint64_t`）编码到单个64位的`gtsam.Key`中。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/LabeledSymbol.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import LabeledSymbol
```

## 初始化

可以通过提供类型字符、标签字符和索引来创建`LabeledSymbol`。也可以通过解码现有`gtsam.Key`（整数）来创建。

```python
# Create LabeledSymbol 'x' from robot 'A' with index 7
# The underlying C++ expects chars, so we have to convert single-character strings with ord()
lsym1 = LabeledSymbol(ord('x'), ord('A'), 7)
print(f"LabeledSymbol from char/label/index: {lsym1}")

# Get the underlying integer key
key1 = lsym1.key()

# Reconstruct LabeledSymbol from the key
# Note: Decoding a key assumes it was encoded as a LabeledSymbol.
# If you decode a standard Symbol key, the label might be garbage.
x0_key = gtsam.Symbol('x', 0).key()
lsym2 = LabeledSymbol(x0_key)
print(f"LabeledSymbol from key {x0_key}: {lsym2}")
```

## 属性和用法

您可以访问类型字符、标签字符、索引和底层整数键。

```python
robotB_landmark = LabeledSymbol(ord('l'), ord('B'), 3)

print(f"LabeledSymbol: {robotB_landmark}")
print(f"  Char (Type): {robotB_landmark.chr()}")
print(f"  Label (Robot): {robotB_landmark.label()}")
print(f"  Index: {robotB_landmark.index()}")
print(f"  Key: {robotB_landmark.key()}")
```

## 简写函数

GTSAM提供了一个方便的简写函数`gtsam.mrsymbol(c, label, j)`。

```python
pc2_key = gtsam.mrsymbol(ord('p'), ord('C'), 2)

print(f"LabeledSymbol(ord('p'), ord('C'), 2).key() == gtsam.mrsymbol(ord('p'), ord('C'), 2): {LabeledSymbol(ord('p'), ord('C'), 2).key() == pc2_key}")
```

## Ordering 排序
# Ordering

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`Ordering`指定了在推断（例如高斯消元、多前沿QR）过程中变量消元的顺序。排序的选择对生成贝叶斯网或贝叶斯树的运算开销和填充(fill-in)（稀疏性）有显著影响。

GTSAM提供了几种自动计算良好排序的算法，如COLAMD和METIS，或者允许您指定自定义排序。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Ordering.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import Ordering
# Need graph types
from gtsam import SymbolicFactorGraph
from gtsam import symbol_shorthand
import graphviz

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建Ordering

排序可以手动创建，也可以从因子图自动计算。

```python
# Manual creation (list of keys)
manual_ordering = Ordering([X(1), L(1), X(2), L(2), X(0)])
manual_ordering.print("Manual Ordering: ")
```

```python
# Automatic creation requires a factor graph
# Let's use a simple SymbolicFactorGraph for structure
graph = SymbolicFactorGraph()
graph.push_factor(X(0))
graph.push_factor(X(0), X(1))
graph.push_factor(X(1), X(2))
graph.push_factor(X(0), L(1))
graph.push_factor(X(1), L(1))
graph.push_factor(X(1), L(2))
graph.push_factor(X(2), L(2))

# COLAMD ordering
colamd_ordering = Ordering.ColamdSymbolicFactorGraph(graph)
colamd_ordering.print("COLAMD Ordering: ")
```

## 自动排序算法：COLAMD vs METIS

GTSAM提供了从因子图自动计算消元排序的算法。两种常见算法是：

1.  **COLAMD（列近似最小度, Column Approximate Minimum Degree）：** 一种贪心算法，旨在每一步消元中最小化*填充(fill-in)*。它通常产生适合顺序执行的稀疏直接方法的排序。
2.  **METIS：** 一种图划分算法。它旨在找到很好地划分图的排序，通常导致更平衡的消元树，这可能有利于并行计算，有时与纯局部贪心方法如COLAMD相比，可以减少总体填充，特别是对于大型结构化问题。

让我们使用2D网格因子图来说明差异。

```python
# Create a 3x4 grid graph
ROWS, COLS = 3, 4

# Use 'x' symbols for grid nodes
X_grid = lambda r, c: X(10 * (r + 1) + c + 1)

def create_grid_graph():
    """Creates a SymbolicFactorGraph representing a 2D grid."""
    graph = SymbolicFactorGraph()
    keys = []
    positions = {}
    for r in range(ROWS):
        for c in range(COLS):
            key = X_grid(r, c)
            positions[key] = gtsam.Point2(c, COLS-r)
            keys.append(key)
            # Add binary factors connecting to right and down neighbors
            if c + 1 < COLS:
                key_right = X_grid(r, c + 1)
                graph.push_factor(key, key_right)
            if r + 1 < ROWS:
                key_down = X_grid(r + 1, c)
                graph.push_factor(key, key_down)
    return graph, keys, positions

grid_graph, grid_keys, positions = create_grid_graph()
```

这是我们网格图的结构。边代表连接变量（节点）的因子。

```python
writer = gtsam.DotWriter(binaryEdges = True)
writer.variablePositions = positions
display(graphviz.Source(grid_graph.dot(writer=writer), engine='neato'))
```

### COLAMD排序和生成的贝叶斯网

现在，我们计算COLAMD排序并按照此顺序消元变量。

```python
# COLAMD (Column Approximate Minimum Degree) ordering
colamd_ordering = Ordering.ColamdSymbolicFactorGraph(grid_graph)
print("COLAMD Ordering: ")
colamd_ordering.print()

# Eliminate using COLAMD ordering
bayes_tree_colamd = grid_graph.eliminateMultifrontal(colamd_ordering)

# Visualize the resulting Bayes Net
print("\nCOLAMD Bayes Net Structure:")
display(graphviz.Source(bayes_tree_colamd.dot()))
```

### METIS排序和生成的贝叶斯网

接下来，我们计算METIS排序并可视化其生成的贝叶斯网。将其结构与COLAMD生成的结构进行比较。

```python
metis_ordering = Ordering.MetisSymbolicFactorGraph(grid_graph)
print("METIS Ordering: ")
metis_ordering.print()

# Eliminate using METIS ordering
bayes_tree_metis = grid_graph.eliminateMultifrontal(metis_ordering)

# Visualize the resulting Bayes Net
print("\nMETIS Bayes Net Structure:")
display(graphviz.Source(bayes_tree_metis.dot()))
```

### 比较

观察COLAMD和METIS生成的贝叶斯树结构的差异：

*   **COLAMD：** 通常产生更"细长"或更深的贝叶斯树。团（贝叶斯树中的条件概率）最初可能较小，但在根部（最后消元的变量）附近可能变大。
*   **METIS：** 倾向于产生更"丛生"或更平衡的树。它试图划分图，在分区内首先消元变量，导致可能更大的初始团，但通常整体结构更浅，分隔符更小（在树的高层连接团的变量）。

何时应该选择其中一个而不是另一个？最佳选择通常取决于具体的问题结构及运算目标：

*   **当以下情况时使用COLAMD：**
    *   您需要快速获得一个好的通用排序。COLAMD*计算排序本身*通常比METIS快得多。
    *   您主要使用*顺序*求解器（在单个CPU核上运行）。COLAMD的贪心策略通常很适合在此场景下最小化填充。
    *   因子图相对较小，或不具有高度规则的结构，使得复杂划分不会带来显著好处。

*   **当以下情况时使用METIS：**
    *   您旨在使用*并行*求解器（例如，使用带有TBB的GTSAM多前沿求解器）获得最佳性能。METIS的图划分方法倾向于创建更平衡的贝叶斯树，从而允许在多个CPU核上更好地分配工作负载。
    *   您正在处理非常大规模的问题，特别是那些具有规则结构的问题（如来自有限元分析的大网格、网格，或广泛的SLAM地图）。在此类问题上，METIS有时可以找到比COLAMD总体*填充*少得多的排序，从而导致更快的因子分解，即使计算排序本身花费更长的时间。
    *   计算排序的成本与随后的因子分解和求解步骤的成本相比可以忽略不计（例如，您为重复求解的结构计算一次排序）。

**总结：** COLAMD是一个鲁棒且快速的默认选择。METIS通常是大规模问题和并行执行的首选，以较慢的排序计算成本，可能提供更好的最终因子分解性能。您可能需要对特定问题类型进行实验以确定最佳选择。

有关COLAMD和METIS的更多信息，请参见[Factor Graphs for Robot Perception](https://www.cs.cmu.edu/~kaess/pub/Dellaert17fnt.pdf)。

### 约束排序

有时，我们希望强制某些变量最后消元（例如，SLAM中当前的机器人位姿）。`Ordering.ColamdConstrainedLast`允许COLAMD实现此功能。

```python
# Example: Constrained COLAMD forcing corners (x0, x24) to be eliminated last
# Note: We use the grid_graph defined earlier
corner_keys = gtsam.KeyVector([X_grid(0, 0), X_grid(ROWS-1, COLS-1)])
constrained_ordering = Ordering.ColamdConstrainedLastSymbolicFactorGraph(grid_graph, corner_keys)
print(f"Constrained COLAMD ({gtsam.DefaultKeyFormatter(corner_keys[0])}, {gtsam.DefaultKeyFormatter(corner_keys[1])} last):")
constrained_ordering.print()
```

## 访问元素

`Ordering`行为类似于一个键的向量。

```python
print(f"COLAMD Ordering size: {colamd_ordering.size()}")

# Access by index
key_at_0 = colamd_ordering.at(0)
print(f"Key at position 0 (COLAMD): {gtsam.DefaultKeyFormatter(key_at_0)}")
```

## 追加键

您可以使用`push_back`向现有排序追加键。

```python
# Use the COLAMD ordering from the grid example
appended_ordering = Ordering(colamd_ordering)
appended_ordering.push_back(L(0)) # Append a landmark key
appended_ordering.push_back(X(ROWS*COLS)) # Append a new pose key x25

appended_ordering.print("Appended Ordering: ")
```

## Shortcuts 快捷方式
# 高效边缘分布计算

GTSAM可以非常高效地计算贝叶斯树中的边缘分布。在本文中，我们说明了用于**缓存**贝叶斯树中条件分布$P(S \mid R)$的"快捷方式(shortcut)"机制，从而支持高效的其他边缘分布查询。我们假设读者熟悉**贝叶斯树**（参见[前一篇文章](#)）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Shortcuts.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

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

## 玩具示例

我们创建一个小型贝叶斯树：

\begin{equation}
P(a \mid b) P(b,c \mid r) P(f \mid e) P(d,e \mid r) P(r).
\end{equation}

下面是使用GTSAM的离散封装来定义和构建相应贝叶斯树的一些Python代码。我们将使用一个离散示例，即创建一个`DiscreteBayesTree`。

```python
from gtsam import DiscreteConditional, DiscreteBayesTree, DiscreteBayesTreeClique, DecisionTreeFactor
```

```python
# Make discrete keys (key in elimination order, cardinality):
keys = [(0, 2), (1, 2), (2, 2), (3, 2), (4, 2), (5, 2), (6, 2)]
names = {0: 'a', 1: 'f', 2: 'b', 3: 'c', 4: 'd', 5: 'e', 6: 'r'}
aKey, fKey, bKey, cKey, dKey, eKey, rKey = keys
keyFormatter = lambda key: names[key]
```

```python
# 1. Root Clique: P(r)
cliqueR = DiscreteBayesTreeClique(DiscreteConditional(rKey, "0.4/0.6"))

# 2. Child Clique 1: P(b, c | r)
cliqueBC = DiscreteBayesTreeClique(
    DiscreteConditional(
        2, DecisionTreeFactor([bKey, cKey, rKey], "0.3 0.7 0.1 0.9 0.2 0.8 0.4 0.6")
    )
)

# 3. Child Clique 2: P(d, e | r)
cliqueDE = DiscreteBayesTreeClique(
    DiscreteConditional(
        2, DecisionTreeFactor([dKey, eKey, rKey], "0.1 0.9 0.9 0.1 0.2 0.8 0.3 0.7")
    )
)

# 4. Leaf Clique from Child 1: P(a | b)
cliqueA = DiscreteBayesTreeClique(DiscreteConditional(aKey, [bKey], "1/3 3/1"))

# 5. Leaf Clique from Child 2: P(f | e)
cliqueF = DiscreteBayesTreeClique(DiscreteConditional(fKey, [eKey], "1/3 3/1"))
```

```python
# Build the BayesTree:
bayesTree = DiscreteBayesTree()

# Insert root:
bayesTree.insertRoot(cliqueR)

# Attach child cliques to root:
bayesTree.addClique(cliqueBC, cliqueR)
bayesTree.addClique(cliqueDE, cliqueR)

# Attach leaf cliques:
bayesTree.addClique(cliqueA, cliqueBC)
bayesTree.addClique(cliqueF, cliqueDE)

# bayesTree.print("bayesTree", keyFormatter)
```

```python
import graphviz
graphviz.Source(bayesTree.dot(keyFormatter))
```

## P(a)的朴素计算
边缘分布$P(a)$可以通过对树中的其他变量求和来计算：
$$
P(a) = \sum_{b, c, d, e, f, r} P(a, b, c, d, e, f, r)
$$

使用贝叶斯树结构，我们有：

$$
P(a) = \sum_{b, c, d, e, f, r} P(a \mid b) P(b, c \mid r) P(f \mid e) P(d, e \mid r) P(r)
$$

但我们可以忽略不在从$a$到根$r$路径上的变量$e$和$f$。确实，由结合律我们有：

$$
P(a) = \sum_{r} \Bigl\{ \sum_{e,f} P(f \mid e) P(d, e \mid r) \Bigr\} \sum_{b, c, d} P(a \mid b) P(b, c \mid r) P(r)
$$

其中分组项对任何$r$的值求和为1，因此：

$$
P(a) = \sum_{r, b, c, d} P(a \mid b) P(b, c \mid r) P(r).
$$

## 通过快捷方式的记忆化

在GTSAM中，我们递归计算此结果。

#### 第1步
我们要通过以下公式计算边缘分布：
$$
P(a) = \sum_{r, b} P(a \mid b) P(b).
$$
其中$P(b)$是该团的分隔符。

#### 第2步
要计算分隔符的边缘分布，我们使用**快捷方式** $P(b|r)$：
$$
P(b) = \sum_{r} P(b \mid r) P(r).
$$
一般而言，快捷方式$P(S|R)$直接将该团的分隔符$S$条件于根团$R$，即使中间有许多其他团。这就是为什么它被称为*快捷方式*。

#### 第3步（可选）
如果快捷方式已经计算过，那么我们就完成了！如果没有，我们递归计算它：
$$
P(S\mid R) = \sum_{F_p,\,S_p \setminus S}P(F_p \mid S_p) P(S_p \mid R).
$$
上面$P(F_p \mid S_p)$是父团，由运行交集性质(running intersection property)我们知道分隔符$S$是父团变量的子集。
注意递归是因为我们可能还没有$P(S_p \mid R)$，因此可能需要依次计算，等等。递归在根以下的节点处结束，**在我们获得$P(S\mid R)$后我们将其缓存**。

在我们的示例中，计算很简单：
$$
P(b|r) = \sum_{c} P(b, c \mid r),
$$
因为此时父分隔符已经是根，所以省略了$P(S_p \mid R)$。这也是递归的终点。

```python
# Marginal of the leaf variable 'a':
print(bayesTree.numCachedSeparatorMarginals())
marg_a = bayesTree.marginalFactor(aKey[0])
print("Marginal P(a):\n", marg_a)
print(bayesTree.numCachedSeparatorMarginals())
```

```python

# Marginal of the internal variable 'b':
print(bayesTree.numCachedSeparatorMarginals())
marg_b = bayesTree.marginalFactor(bKey[0])
print("Marginal P(b):\n", marg_b)
print(bayesTree.numCachedSeparatorMarginals())
```

```python

# Joint of leaf variables 'a' and 'f': P(a, f)
# This effectively needs to gather info from two different branches
print(bayesTree.numCachedSeparatorMarginals())
marg_af = bayesTree.jointBayesNet(aKey[0], fKey[0])
print("Joint P(a, f):\n", marg_af)
print(bayesTree.numCachedSeparatorMarginals())
```

## Symbol 符号
# Symbol

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`Symbol`是一个GTSAM类，用于为变量创建语义上有意义的键（`gtsam.Key`）。它将一个字符（`unsigned char`）和一个索引（`uint64_t`）组合成一个64位整数键。这允许轻松识别变量类型及其索引，例如，'x'用于位姿，'l'用于路标(landmark)，如`x0`、`x1`、`l0`。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/Symbol.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import Symbol
```

## 初始化

可以通过提供字符和索引创建`Symbol`。也可以通过解码现有`gtsam.Key`（整数）来创建。

```python
# Create Symbol 'x' with index 5
sym1 = Symbol('x', 5)
print(f"Symbol from char/index: {sym1.string()}")

# Get the underlying integer key
key1 = sym1.key()

# Reconstruct Symbol from the key
sym2 = Symbol(key1)
print(f"Symbol from key {key1}: {sym2.string()}")
```

## 属性和用法

您可以访问字符、索引和底层整数键。

```python
landmark_sym = Symbol('l', 10)

print(f"Symbol: {landmark_sym.string()}")
print(f"  Char: {landmark_sym.chr()}")
print(f"  Index: {landmark_sym.index()}")
print(f"  Key: {landmark_sym.key()}")

# Symbols are often used directly where Keys are expected in GTSAM functions,
# as they implicitly convert.
# e.g., values.insert(landmark_sym, gtsam.Point3(1,2,3))
```

## 简写函数

GTSAM提供了方便的简写函数`X(j)`、`L(j)`等，它们等效于`gtsam.Symbol('x', j)`、`gtsam.Symbol('l', j)`。要使用这些，首先通过`from gtsam import symbol_shorthand`导入，然后使用`X = symbol_shorthand.X`等语句设置要使用的变量。

```python
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L

print(f"Symbol('x', 0).key() == X(0): {Symbol('x', 0).key() == X(0)}")
print(f"Symbol('l', 1).key() == L(1): {Symbol('l', 1).key() == L(1)}")
```

## VariableIndex 变量索引
# VariableIndex

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`VariableIndex`提供了一种高效的方式来查找`FactorGraph`中哪些因子涉及特定变量（键）。它为每个变量存储包含该变量的因子索引的列表。

此结构通常由GTSAM算法（如排序方法或消元）内部计算，但如果需要，也可以显式创建，例如，当多个操作需要此信息时以提高性能。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/inference/doc/VariableIndex.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import VariableIndex, Ordering
# Need a graph type for creation
from gtsam import SymbolicFactorGraph
from gtsam import symbol_shorthand

X = symbol_shorthand.X
L = symbol_shorthand.L
```

## 创建VariableIndex

`VariableIndex`通常从现有的`FactorGraph`创建。

```python
# Create a simple SymbolicFactorGraph
graph = SymbolicFactorGraph()
graph.push_factor(X(0))           # Factor 0
graph.push_factor(X(0), X(1))     # Factor 1
graph.push_factor(X(1), X(2))     # Factor 2
graph.push_factor(X(0), L(1))     # Factor 3
graph.push_factor(X(1), L(1))     # Factor 4
graph.push_factor(X(1), L(2))     # Factor 5
graph.push_factor(X(2), L(2))     # Factor 6

# Create VariableIndex from the graph
variable_index = VariableIndex(graph)

# Print the index
variable_index.print("VariableIndex: ")
```

## 访问信息

您可以查询变量数量、因子数量、条目数量，并查找与特定变量关联的因子。

```python
print(f"Number of variables (size): {variable_index.size()}")
print(f"Number of factors (nFactors): {variable_index.nFactors()}")
print(f"Number of variable-factor entries (nEntries): {variable_index.nEntries()}")

# Get factors involving a specific variable
factors_x1 = variable_index.at(X(1)) # Returns a FactorIndices (FastVector<size_t>)
print(f"Factors involving x1: {factors_x1}")

# Use key directly
factors_l1 = variable_index.at(L(1))
print(f"Factors involving l1: {factors_l1}")
```

## 在算法中的使用

`VariableIndex`主要用作其他算法的输入，特别是`Ordering.Colamd`等排序方法。

```python
# Compute COLAMD ordering directly from the VariableIndex
colamd_ordering = Ordering.Colamd(variable_index)
colamd_ordering.print("COLAMD Ordering from VariableIndex: ")
```

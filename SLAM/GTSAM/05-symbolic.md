# Symbolic 符号模块

GTSAM中的`symbolic`模块处理因子图(factor graph)和贝叶斯网(Bayesian network)的*结构*，与因子的具体数值类型（如高斯或离散）无关。它允许分析图连接性、确定最优变量消元顺序，以及理解推断结果对象的稀疏结构。

这对于高效推断至关重要，因为符号消元步骤决定了数值因子分解的计算复杂度和内存需求。

这里的类主要用于*说明*符号消元。在内部，GTSAM确实分析其他因子图类型的结构，但不会显式地转换为符号因子图。


## 类(Classes)

以下是`symbolic`模块中关键类的概述：

### 因子图

-   **[SymbolicFactor](doc/SymbolicFactor.ipynb)**：表示因子图中一组变量（键）之间的连接性，不关联任何具体的数值函数。它定义了图的超边。
-   **[SymbolicFactorGraph](doc/SymbolicFactorGraph.ipynb)**：`SymbolicFactor`对象的集合，表示因子图的整体结构。

### 消元产物

这些类表示符号变量消元的结果：

-   **[SymbolicConditional](doc/SymbolicConditional.ipynb)**：表示条件概率分布P(Frontals | Parents)的结构。它存储从一组因子中消元一个或多个变量所产生的前沿（被条件化）变量和父变量的键。
-   **[SymbolicBayesNet](doc/SymbolicBayesNet.ipynb)**：由`SymbolicConditional`对象组成的有向无环图，表示因子化分布P(X) = Π P(Xi | Parents(Xi))的结构。通常由顺序消元产生。
-   **[SymbolicBayesTree](doc/SymbolicBayesTree.ipynb)**：一棵树结构，其中每个节点（`SymbolicBayesTreeClique`）表示一个`SymbolicConditional` P(Frontals | Separator)。这是多前沿消元的结果，是高效增量更新(iSAM)和精确边缘分布计算的基础结构。
-   **[SymbolicBayesTreeClique](doc/SymbolicBayesTreeClique.ipynb)**：表示`SymbolicBayesTree`中的单个团(clique)（节点），包含一个`SymbolicConditional`。

### 消元结构

这些类表示消元过程中使用的中间结构：

-   **[SymbolicEliminationTree](doc/SymbolicEliminationTree.ipynb)**：表示顺序变量消元的依赖结构。每个节点对应于消元单个变量。
-   **[SymbolicJunctionTree](doc/SymbolicJunctionTree.ipynb)**（团树, Clique Tree）：多前沿消元中使用的中间结构，从消元树推导而来。每个节点（`SymbolicCluster`）表示一组一起消元的变量的团，并存储与该团关联的*因子*。

**重要性**
执行符号分析可用于：
1. 选择最优排序(Ordering)：找到最小化填充(fill-in，消元过程中创建的新连接)的排序是高效数值因子分解的关键。
2. 预测计算成本：贝叶斯网或贝叶斯树的结构决定了后续数值操作（如求解或边缘化）的复杂度。
3. 内存分配：了解结构允许为数值因子分解预分配内存。
符号模块为GTSAM的高效推断算法提供了基础。

## SymbolicBayesNet
# SymbolicBayesNet

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicBayesNet`是由`SymbolicConditional`对象组成的有向无环图(DAG)。它纯粹用变量连接性来表示因子化概率分布P(X) = Π P(Xi | Parents(Xi))的结构。

它通常是对`SymbolicFactorGraph`运行顺序变量消元的结果。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicBayesNet.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicConditional, SymbolicFactorGraph, Ordering
from gtsam.symbol_shorthand import X, L
import graphviz
```

## 创建SymbolicBayesNet

SymbolicBayesNet通常通过消元[SymbolicFactorGraph](SymbolicFactorGraph.ipynb)来创建。但您也可以直接构建：

```python
from gtsam import SymbolicBayesNet

# Create a new Bayes Net
symbolic_bayes_net = SymbolicBayesNet()

# Add conditionals directly
symbolic_bayes_net.push_back(SymbolicConditional(L(1), X(0)))  # P(l1 | x0)
symbolic_bayes_net.push_back(SymbolicConditional(X(0), X(1)))  # P(x0 | x1)
symbolic_bayes_net.push_back(SymbolicConditional(L(2), X(1)))  # P(l2 | x1)
symbolic_bayes_net.push_back(SymbolicConditional(X(1), X(2)))  # P(x1 | x2)
symbolic_bayes_net.push_back(SymbolicConditional(X(2)))  # P(x2)

symbolic_bayes_net.print("Directly Built Symbolic Bayes Net:\n")
```

## 访问条件概率和可视化

```python
# Access a conditional by index
conditional_1 = bayes_net.at(1) # P(x0 | l1)
conditional_1.print("Conditional at index 1: ")

# Visualize the Bayes Net structure
display(graphviz.Source(bayes_net.dot()))
```

## SymbolicBayesTree
# SymbolicBayesTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicBayesTree`是一棵树结构，其中每个节点（`SymbolicBayesTreeClique`）表示在多前沿消元过程中一起消元的变量团。与存储因子的`SymbolicJunctionTree`不同，`SymbolicBayesTree`为每个团存储生成的`SymbolicConditional`。

它表示因子化结构$P(X) = Π P(C_j | S_j)$，其中$C_j$是团$j$的前沿变量，$S_j$是其分隔符变量（贝叶斯树中的父节点）。这种结构在计算上对推断很有利，特别是用于计算边缘分布或执行增量更新（如在iSAM中）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicBayesTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicFactorGraph, Ordering
from gtsam.symbol_shorthand import X, L
import graphviz # For visualization
```

## 创建SymbolicBayesTree

贝叶斯树通常通过使用多前沿方法消元`SymbolicFactorGraph`来创建。

```python
# Create a factor graph from a GTSFM problem
graph = SymbolicFactorGraph()

edges = [(2, 4), (2, 5), (2, 27), (4, 6), (5, 6), (5, 7), (5, 8), (5, 9), (6, 7), (6, 8), (6, 9),
         (7, 8), (7, 9), (8, 9), (9, 12), (9, 24), (9, 28), (10, 12), (10, 29), (20, 21), (20, 22),
         (20, 23), (20, 24), (21, 22), (21, 23), (21, 24), (22, 23), (22, 24), (23, 24), (25, 26),
         (25, 27), (25, 28), (25, 29), (26, 27), (26, 28), (26, 29), (27, 28), (27, 29), (28, 29)]

for i,j in edges:
    graph.push_factor(X(i), X(j))

# Define an elimination ordering
ordering = Ordering.MetisSymbolicFactorGraph(graph)  # Use METIS for this example

# Eliminate using Multifrontal method
bayes_tree = graph.eliminateMultifrontal(ordering)
```

## 可视化和属性

贝叶斯树结构可以可视化，我们可以查询其属性。

```python
print(f"\nBayes Tree size (number of cliques): {bayes_tree.size()}")
print(f"Is the Bayes Tree empty? {bayes_tree.empty()}")
```

```python
# Visualize the Bayes tree using graphviz
display(graphviz.Source(bayes_tree.dot()))
```

## 遍历贝叶斯树

```python
roots = bayes_tree.roots()
print(f"Number of roots: {len(roots)}")

def traverse_clique(clique):
    if clique:
        clique.print("\nClique:\n")
        for j in range(clique.nrChildren()):
            traverse_clique(clique[j])

for root in roots:
    traverse_clique(root)
```

## SymbolicBayesTreeClique
# SymbolicBayesTreeClique

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicBayesTreeClique`表示`SymbolicBayesTree`中的一个节点。每个团对应在多前沿消元过程中一起消元的一组变量（前沿变量）。

团的关键方面：
*   **条件概率(Conditional)：** 它存储消元前沿变量所产生的`SymbolicConditional` P(Frontals | Parents/Separator)。
*   **树结构：** 它维护指向贝叶斯树中父团和子团的指针，定义树的拓扑。
*   **前沿变量和分隔符变量：** 隐式定义前沿变量（在此团中消元）和分隔符变量（贝叶斯树中的父节点，将其连接到父团）。

用户通常作为整体与`SymbolicBayesTree`交互，但理解团结构有助于理解贝叶斯树如何表示因子化分布并促进高效推断。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicBayesTreeClique.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicFactorGraph, Ordering
# SymbolicBayesTreeClique is accessed *through* a SymbolicBayesTree
from gtsam.symbol_shorthand import X, L
import graphviz
```

## 访问和检查团

团通过首先创建`SymbolicBayesTree`（通常通过消元）然后访问其节点来获得。

```python
# Create a factor graph
graph = SymbolicFactorGraph()
graph.push_factor(X(0))
graph.push_factor(X(0), X(1))
graph.push_factor(X(1), X(2))
graph.push_factor(X(0), L(1))
graph.push_factor(X(1), L(2))

# Eliminate to get a Bayes Tree
ordering = Ordering.ColamdSymbolicFactorGraph(graph)
bayes_tree = graph.eliminateMultifrontal(ordering)

print(f"Bayes Tree has {bayes_tree.size()} cliques.")

roots = bayes_tree.roots()
print(f"Number of roots: {len(roots)}")

# Visualize the Bayes tree using graphviz
display(graphviz.Source(bayes_tree.dot()))
```

```python
def inspect(clique):
    print("\nInspecting Clique 0:")
    clique.print("  Clique Structure: ")

    # Get the conditional stored in the clique
    conditional = clique.conditional()
    if conditional:
        conditional.print("  Associated Conditional: P(")
    else:
        print("  Clique has no associated conditional (might be empty root)")

    # Check properties
    print(f"  Is Root? {clique.isRoot()}")
    # Accessing parent/children is possible in C++ but might be less direct or typical in Python wrapper usage
    # Parent clique (careful, might be null for root)
    parent_clique = clique.parent()
    if parent_clique:
        print("  Parent Clique exists.")
    else:
        print("  Parent Clique is None (likely a root clique).")

    print(f"  Number of Children: {clique.nrChildren()}") # Example if method existed

def traverse_clique(clique):
    inspect(clique)
    for j in range(clique.nrChildren()):
        traverse_clique(clique[j])

for root in roots:
    traverse_clique(root)
```

## SymbolicConditional
# SymbolicConditional

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicConditional`以纯符号形式表示条件概率分布P(Frontals | Parents)。它仅存储前沿变量和父变量的键，不关联任何数值概率函数。

`SymbolicConditional`对象通常作为对`SymbolicFactorGraph`进行符号消元的结果产生。这些对象的集合形成`SymbolicBayesNet`。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicConditional.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicConditional
from gtsam.symbol_shorthand import X, L
```

## 创建SymbolicConditional

条件概率指定前沿（被条件化）变量和父变量。

```python
# P(X(0))
c0 = SymbolicConditional(X(0))
c0.print("P(X(0)): ")
print(f" Nr Frontals: {c0.nrFrontals()}, Nr Parents: {c0.nrParents()}\n")

# P(X(1) | X(0))
c1 = SymbolicConditional(X(1), X(0))
c1.print("P(X(1) | X(0)): ")
print(f" Nr Frontals: {c1.nrFrontals()}, Nr Parents: {c1.nrParents()}\n")

# P(X(2) | X(0), X(1))
c2 = SymbolicConditional(X(2), X(0), X(1))
c2.print("P(X(2) | X(0), X(1)): ")
print(f" Nr Frontals: {c2.nrFrontals()}, Nr Parents: {c2.nrParents()}\n")

# P(L(1) | X(0), X(1))
c3 = SymbolicConditional(L(1), X(0), X(1))
c3.print("P(L(1) | X(0), X(1)): ")
print(f" Nr Frontals: {c3.nrFrontals()}, Nr Parents: {c3.nrParents()}\n")

# Create from keys and number of frontals, e.g. P(X(2), L(1) | X(0), X(1))
# Keys = [Frontals..., Parents...]
c4 = SymbolicConditional.FromKeys([X(2), L(1), X(0), X(1)], 2)
c4.print("P(X(2), L(1) | X(0), X(1)): ")
print(f" Nr Frontals: {c4.nrFrontals()}, Nr Parents: {c4.nrParents()}")
```

## SymbolicEliminationTree
# SymbolicEliminationTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicEliminationTree`表示变量消元中使用的计算结构，特别是在稀疏Cholesky或QR因子分解中。树中的每个节点对应单个变量的消元。

树结构揭示了依赖关系：变量（节点）的消元依赖于其子节点在树中的结果。树的根对应于最后被消元的变量。此结构与生成的贝叶斯网或贝叶斯树密切相关。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicEliminationTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicEliminationTree, SymbolicFactorGraph, Ordering
from gtsam.symbol_shorthand import X, L
```

## 创建SymbolicEliminationTree

消元树从`SymbolicFactorGraph`和`Ordering`构造。

```python
# Create a factor graph
graph = SymbolicFactorGraph()
graph.push_factor(X(0))
graph.push_factor(X(0), X(1))
graph.push_factor(X(1), X(2))
graph.push_factor(X(0), L(1))
graph.push_factor(X(1), L(2))

# Define an elimination ordering
ordering = Ordering([L(1), L(2), X(0), X(1), X(2)])  # Eliminate L(1) first, then X(0), X(1), X(2) last

# Construct the elimination tree
elimination_tree = SymbolicEliminationTree(graph, ordering)

# Print the tree structure (text representation)
elimination_tree.print("Symbolic Elimination Tree:\n")
```

*(注意：消元树结构的直接可视化不能通过简单的`.dot()`方法实现（不像因子图或贝叶斯网/树在Python封装器中那样），但print输出显示了父子关系。)*

## SymbolicFactor
# SymbolicFactor

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicFactor`表示因子图中变量之间的连接性（拓扑），不关联任何具体的数值函数。它主要用于*说明*符号消元。在内部，GTSAM确实分析其他因子图类型的结构，但不会显式地转换为符号因子图。

它继承自`gtsam.Factor`，存储它连接的变量的键（索引）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
import gtsam
from gtsam import SymbolicFactor
from gtsam.symbol_shorthand import X

# Example Keys
x0, x1, x2 = X(0), X(1), X(2)
```

## 创建SymbolicFactor

可以通过指定它们涉及的键来创建SymbolicFactor。

```python
# Unary factor
f0 = SymbolicFactor(x0)
f0.print("Unary Factor f0: \n")

# Binary factor
f1 = SymbolicFactor(x0, x1)
f1.print("Binary Factor f1: \n")

# Ternary factor
f2 = SymbolicFactor(x1, x2, x0)
f2.print("Ternary Factor f2: \n")

# From a list of keys
keys = gtsam.KeyVector([x0, x1, x2])
f3 = SymbolicFactor.FromKeys(keys)
f3.print("Factor f3 from KeyVector: \n")
```

## SymbolicFactorGraph
# SymbolicFactorGraph

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicFactorGraph`是一个纯粹由`SymbolicFactor`对象组成的因子图。它表示概率图模型（如马尔可夫随机场, Markov Random Field）的结构或拓扑，不包含数值细节。

它主要用于*说明*符号消元，符号消元确定生成的贝叶斯网或贝叶斯树的结构，并找到高效的变量消元排序（例如，使用COLAMD或METIS）。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicFactorGraph.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicFactorGraph, Ordering
from gtsam.symbol_shorthand import X, L
import graphviz
```

## 创建和操作符号因子图

```python
# Create an empty graph
graph = SymbolicFactorGraph()

# Add factors (using convenience methods)
graph.push_factor(X(0)) # Unary
graph.push_factor(X(0), X(1))
graph.push_factor(X(1), X(2))
graph.push_factor(X(0), L(1))
graph.push_factor(X(1), L(2))

# Print the graph structure
graph.print("Symbolic Factor Graph:\n")

# Visualize the graph using Graphviz
graphviz.Source(graph.dot())
```

## 符号消元

我们可以执行符号消元以获得生成的贝叶斯网或贝叶斯树的结构。

```python
# Define an elimination ordering (can also be computed automatically)
ordering = Ordering([L(1), L(2), X(0), X(1), X(2)])
```

```python
# Eliminate sequentially to get a Bayes Net structure
bayes_net = graph.eliminateSequential(ordering)
bayes_net.print("\nResulting Symbolic Bayes Net:\n")

# Visualize the Bayes Net using Graphviz
graphviz.Source(bayes_net.dot())
```

```python
# Eliminate using Multifrontal method to get a Bayes Tree structure
bayes_tree = graph.eliminateMultifrontal(ordering)
bayes_tree.print("\nResulting Symbolic Bayes Tree:\n")

# Visualize the Bayes Tree using Graphviz
graphviz.Source(bayes_tree.dot())
```

## SymbolicJunctionTree
# SymbolicJunctionTree

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

`SymbolicJunctionTree`（在GTSAM文档中常与团树Clique Tree互换使用）是多前沿变量消元中使用的数据结构。它从`EliminationTree`推导而来。

与消元树的关键区别：
*   **节点是团：** 联结树中的每个节点表示一个*团*（一组一起消元的变量），而不仅仅是单个变量。
*   **存储因子：** 节点（`SymbolicCluster`对象）存储与该团中变量关联的符号因子。

联结树组织因子和变量以进行高效的多前沿消元，该消元在这些较大的团中同时处理变量。消元联结树的结果是一个`SymbolicBayesTree`。

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/symbolic/doc/SymbolicJunctionTree.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

```python
%pip install --quiet gtsam-develop
```

```python
from gtsam import SymbolicJunctionTree, SymbolicEliminationTree, SymbolicFactorGraph, Ordering
from gtsam.symbol_shorthand import X, L
```

## 创建SymbolicJunctionTree

联结树从`SymbolicEliminationTree`构造。

```python
# Create a factor graph
graph = SymbolicFactorGraph()
graph.push_factor(X(0))
graph.push_factor(X(0), X(1))
graph.push_factor(X(1), X(2))
graph.push_factor(X(0), L(1))
graph.push_factor(X(1), L(2))

# Define an elimination ordering
ordering = Ordering([L(1), L(2), X(0), X(1), X(2)])

# Construct the elimination tree first
elimination_tree = SymbolicEliminationTree(graph, ordering)

# Construct the junction tree from the elimination tree
junction_tree = SymbolicJunctionTree(elimination_tree)

# Print the tree structure (text representation)
junction_tree.print("Symbolic Junction Tree:\n")
```

*(注意：`SymbolicCluster`类表示联结树中的节点，包含该团的因子和前沿/分隔符键。直接可视化通常通过生成的贝叶斯树来进行。)*

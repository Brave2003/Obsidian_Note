# Base 基础模块

base模块提供了构成GTSAM基础的基本数据结构、工具函数和数学运算。以下是主要的头文件：

## 核心数学概念
用于定义群(group)、流形(manifold)和李群(Lie group)类的工具，以及这些概念的检查。
- [Manifold.h](Manifold.h) - 定义了具有retract/local操作的可微流形(differentiable manifold)概念。参见[Manifold](doc/Manifold.md)文档。
- [Lie.h](Lie.h) - 实现了将群操作与流形结构相结合的李群概念。参见[LieGroup](doc/LieGroup.md)文档。
- [MatrixLieGroup.h](MatrixLieGroup.h) - 具有Hat/Vee算子的矩阵李群(SO(n), SE(n), SL(n)等)基类。参见[MatrixLieGroup](doc/MatrixLieGroup.md)文档。

以及较少使用的：
- [Group.h](Group.h) - 定义了具有单位元(identity)、逆元(inverse)和复合(compose)操作的群概念。参见[Group](doc/Group.md)文档。
- [VectorSpace.h](VectorSpace.h) - 定义了向量空间(vector space)概念。参见[VectorSpace](doc/VectorSpace.md)文档。
- [ProductLieGroup.h](ProductLieGroup.h) - 李群的笛卡尔积
- [lieProxies.h](lieProxies.h) - 用于测试的代理函数

## 线性代数
针对机器人应用优化的线性代数操作。`Vector`和`Matrix`本质上是Eigen的typedef和封装：
- [Vector.h](Vector.h) - 向量操作和数学函数
- [Matrix.h](Matrix.h) - 基于Eigen构建的基本矩阵操作和线性代数工具
- [cholesky.h](cholesky.h) - 用于求解线性系统的Cholesky分解算法

以下用于Jacobian和Hessian因子：
- [VerticalBlockMatrix.h](VerticalBlockMatrix.h) - 用于高效稀疏操作的特殊块矩阵结构
- [SymmetricBlockMatrix.h](SymmetricBlockMatrix.h) - 用于优化问题的对称块矩阵实现

## Jacobians
- [numericalDerivative.h](numericalDerivative.h) - 用于自动导数计算的数值微分工具
- [OptionalJacobian.h](OptionalJacobian.h) - 用于优化效率的可选Jacobian计算封装

## 容器和数据结构
- 高性能容器类型（FastSet, FastMap等），针对GTSAM优化（对于较新的C++编译器是否仍然相关尚不明确）
- [DSFMap.h](DSFMap.h) - 使用map实现的并查集(Disjoint Set Forest)，用于union-find操作
- [DSFVector.h](DSFVector.h) - 使用vector实现的并查集，性能更好
- [TreeTraversal.h](TreeTraversal.h) - 用于因子图(factor graph)的树遍历算法和工具
- [ConcurrentMap.h](ConcurrentMap.h) - 线程安全的并发map实现

## 调试和开发工具
- [debug.h](debug.h) - 调试工具、断言宏和条件编译标志
- [timing.h](timing.h) - 用于性能测试的时间测量和分析工具
- [TestableAssertions.h](TestableAssertions.h) - 用于单元测试的断言宏，支持容差检查
- [chartTesting.h](chartTesting.h) - 用于测试流形坐标卡(chart)操作的工具

## 采样和统计
- [WeightedSampler.h](WeightedSampler.h) - 用于蒙特卡洛方法的加权随机采样工具
- [sampler.h](sampler.h) - 通用采样工具和随机数生成

## 图算法
- [Kruskal.h](Kruskal.h) - 用于因子图中最小生成树的Kruskal算法
- [BTree.h](BTree.h) - 用于高效存储和检索的平衡树实现

## 类型系统和特征
- [types.h](types.h) - 常见类型定义、别名和前向声明
- [concepts.h](concepts.h) - 用于类型检查和约束的C++概念(concept)定义
- [Testable.h](Testable.h) - 可测试相等性的对象的基类和概念

## 序列化支持
- [SerializationBase.h](SerializationBase.h) - 用于保存/加载对象的基础序列化功能
- [StdOptionalSerialization.h](StdOptionalSerialization.h) - 对std::optional类型的序列化支持

## 内存管理和性能
- [utilities.h](utilities.h) - 通用工具函数、内存管理辅助函数和常见操作
- [FastDefaultAllocator.h](FastDefaultAllocator.h) - 用于关键路径性能提升的自定义分配器
- [Value.h](Value.h) - 用于异构容器的类型擦除值封装

## 模板元编程
- [treeTraversal-inst.h](treeTraversal-inst.h) - 树遍历算法的模板实例化
- [concepts.h](concepts.h) - SFINAE工具和类型特征辅助函数

## Group 群
# 创建`Group`的指南

本指南说明创建满足GTSAM中`Group`概念的类的最低要求。`Group`是一个基础代数概念，先于几何`Manifold`概念。它定义了具有二元操作（复合）、单位元和每个元素逆元的类型。

与其他GTSAM概念一样，这是通过**基于特征(traits)的元编程**实现的，而非类继承。参见[GTSAM-Concepts](GTSAM-Concepts.md)。

### 1. 群操作

本质上，群由四个抽象操作定义：`Identity`、`Compose`、`Between`和`Inverse`。GTSAM根据实现这些操作所使用的C++语法，区分两种"风格(flavor)"的群。

### 2. 实现要求：选择一种风格

您必须实现两种风格之一的方法。

#### A) `MultiplicativeGroup`风格（乘法群）

这适用于使用`operator*`进行复合的群，如旋转或位姿类型。您的类必须提供：

*   静态`Identity`方法。
*   用于复合的`operator*`。
*   `inverse`方法。

#### B) `AdditiveGroup`风格（加法群）

这适用于行为类似向量空间的群，使用`operator+`进行复合。您的类必须提供：

*   静态`Identity`方法。
*   用于复合的`operator+`。
*   用于`Between`操作的二元`operator-`。
*   用于`Inverse`操作的一元`operator-`。

### 3. 前提条件：`Testable`概念

`Group`概念要求先满足`Testable`概念。在满足群要求之前，您的类还必须实现：

*   `print`方法。
*   `equals`方法。

### 4. 将类链接到GTSAM（`traits`特化）

这是将您的类注册到GTSAM的关键步骤。您必须为您的类特化`gtsam::traits`结构体。该特化应继承自`Group.h`中提供的辅助类型之一。辅助类型会自动将您选择的C++运算符映射到抽象的`Compose`、`Between`和`Inverse`操作。

具体做法是，在`gtsam`命名空间中为`traits<MyGroup>`添加一个特化。如果您的群是乘法群，该特化将继承自`internal::MultiplicativeGroup<MyGroup>`。如果您的群是加法群，它将继承自`internal::AdditiveGroup<MyGroup>`。

### 5. 概念检查

要验证您的实现是否正确，请在单元测试文件中添加`GTSAM_CONCEPT_GROUP_INST`宏，后跟类名（用括号括起来）。如果未满足`Group`（包括`Testable`）的任何要求，这将导致编译时错误。

## LieGroup 李群
# 创建`LieGroup`

本指南说明如何创建`LieGroup`类，它建立在`Manifold`概念之上。**请先阅读[Manifold.md](Manifold.md)**，因为它解释了GTSAM中使用的基础特征设计和概念检查。

李群(Lie group)是一个既是群(group)也是流形(manifold)的空间，具有平滑的群操作。在GTSAM中，这意味着它具有`Manifold`的所有性质，外加复合、单位元和逆元的概念。GTSAM使用[奇异递归模板模式(Curiously Recurring Template Pattern, CRTP)](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)来自动提供许多更复杂的流形和群方法。

### 1. 从Manifold到LieGroup

李群有一个特殊元素：**单位元(identity element)**。这允许我们定义以该单位元为中心的全局`Expmap`和`Logmap`操作。`Manifold`概念所需的`retract`和`localCoordinates`方法就是通过这些以单位元为中心的操作来实现的。

要实现一个李群，需要继承`gtsam::LieGroup<MyGroup, D>`。

### 2. `MyGroup`的最低实现要求

#### `MyGroup.h` - 头文件要求

1.  **类定义和继承**：
    *   公开继承自`gtsam::LieGroup<MyGroup, D>`，其中`D`是群切空间的维度。

2.  **基本Typedef和常量**：
    *   `static const size_t dimension = D;`
    *   `using ChartJacobian = gtsam::OptionalJacobian<D, D>;`

3.  **构造函数**：
    *   初始化到单位元的默认构造函数。

4.  **群原语**：
    *   `static MyGroup Identity();`
    *   `MyGroup inverse() const;`
    *   `MyGroup operator*(const MyGroup& other) const;`

5.  **李群原语**：
    *   `static MyGroup Expmap(const gtsam::VectorD& v, ChartJacobian H = {});`：在*单位元处*的指数映射(Exponential map)。必须支持可选的导数。
    *   `static gtsam::VectorD Logmap(const MyGroup& p, ChartJacobian H = {});`：在*单位元处*的对数映射(Logarithm map)。必须支持可选的导数。
    *   `gtsam::MatrixD AdjointMap() const;`：计算伴随映射(Adjoint map)。
    *   `ChartAtOrigin`结构体：此嵌套结构实现了一阶`retract`和`local`。您必须定义：
        *   `static MyGroup Retract(const gtsam::VectorD& v, ChartJacobian H = {});`
        *   `static gtsam::VectorD Local(const MyGroup& p, ChartJacobian H = {});`

6.  **`using`声明**：
    *   `using LieGroup<MyGroup, D>::inverse;`：使基类的`inverse(ChartJacobian H)`方法可见。

7.  **实用工具（`Testable`概念）**：
    *   `void print(const std::string& s) const;`
    *   `bool equals(const MyGroup& other, double tol) const;`

#### 处理动态大小李群

如果您的李群可以在运行时改变大小，您必须做两处相应的修改：

1.  在`LieGroup`模板参数中将静态维度设为`Eigen::Dynamic`：
    *   `class MyGroup : public gtsam::LieGroup<MyGroup, Eigen::Dynamic> { ... };`
2.  您**还必须**提供一个返回对象运行时维度的实例方法：
    *   `int dim() const;`

`LieGroup`框架依赖此方法在运行时正确设置Jacobian和其他矩阵的大小。

### 3. 通过CRTP免费获得的功能

通过继承`gtsam::LieGroup`并定义上述原语，您的类会自动获得以下所有方法的正确实现，包括它们的Jacobian版本。**请勿自己实现这些方法**，除非您*确定*能够更高效（且正确地）实现这些Jacobian。

*   `compose(const MyGroup& other)`：将`*this`与`other`复合。
*   `between(const MyGroup& other)`：计算从`*this`到`other`的相对变换。
*   `retract(const VectorD& v)`：实现为`compose(ChartAtOrigin::Retract(v))`。
*   `localCoordinates(const MyGroup& other)`：实现为`ChartAtOrigin::Local(between(other))`。
*   `expmap(const VectorD& v)`：实现为`compose(Expmap(v))`。
*   `logmap(const MyGroup& other)`：实现为`Logmap(between(other))`。

### 4. Traits和概念检查

`LieGroup`概念建立在`Manifold`之上。您无需自己特化traits结构体，因为`LieGroup`基类会处理。

要验证您的实现，请在单元测试文件中使用`GTSAM_CONCEPT_LIE_INST`宏。

```cpp
#include <gtsam/base/TestableAssertions.h>
#include <gtsam/base/Lie.h>

// Your class include
#include <gtsam/geometry/MyGroup.h>

// This macro will fail to compile if MyGroup is not a valid LieGroup.
GTSAM_CONCEPT_LIE_INST(MyGroup)

// ... your unit tests ...
```

## Manifold 流形
# 创建`Manifold`类

本指南说明在GTSAM中创建新的`Manifold`类的最低要求。流形(manifold)是GTSAM中表示非线性空间的最基础概念。它也是`sLieGroup`等更专门类型的基类。

GTSAM使用两种设计模式的组合：

1.  **基于特征(traits)的元编程**：GTSAM不需要您的类继承自特定的基类，而是使用一个`traits`辅助结构体。您为您的类特化这个`traits`结构体，以"教会"GTSAM的泛型算法如何与其交互。
2.  **概念检查(Concept Checking)**：GTSAM提供在编译时执行检查的宏，以确保您的类正确实现了概念（如`Manifold`）的所有要求。

有关这些机制的更多细节，请参见[GTSAM-Concepts](GTSAM-Concepts.md)。

### 1. 核心思想：Retraction和局部坐标

流形由两个必须互为逆操作的基本操作定义：

*   **`retract(const TangentVector& v) const`**：此操作将向量`v`从当前点(`*this`)的切空间(tangent space)映射回流形上，生成流形上的一个新点。它是向量加法的推广。
*   **`localCoordinates(const ManifoldType& other) const`**：此操作计算将当前点(`*this`, 原点)连接到流形上另一点`other`的切空间向量。它是向量减法的推广。

这些操作必须满足不变性：`a.retract(a.localCoordinates(b))`应约等于`b`。

### 2. 最低实现要求

要创建一个新类`MyManifold`，您必须提供以下组件。请注意，您的类本身不继承任何东西。

#### `MyManifold.h` - 头文件要求

1.  **类定义**：
    *   定义您的类`MyManifold`。它应包含表示其状态所需的数据成员。

2.  **基本Typedef和常量**：
    *   `static const int dimension = D;`
    *   `using TangentVector = gtsam::VectorD;`
    *   `using ChartJacobian = gtsam::OptionalJacobian<D, D>;`

3.  **核心流形方法**：
    *   `Dim()`返回正整数或Eigen::Dynamic
    *   `dim()`返回固定大小或动态大小，如果dimension==Eigen::Dynamic
    *   `MyManifold retract(const TangentVector& v, ChartJacobian H_this = {}, ChartJacobian H_v = {}) const;`
    *   `TangentVector localCoordinates(const MyManifold& other, ChartJacobian H_this = {}, ChartJacobian H_other = {}) const;`

4.  **`Testable`概念工具**：
    *   `Manifold`概念要求先满足`Testable`概念。您必须实现：
        *   `void print(const std::string& s = "") const;`
        *   `bool equals(const MyManifold& other, double tol = 1e-9) const;`

5.  **GTSAM Traits特化**：
    *   在您的类定义下方，您必须特化`gtsam::traits`结构体。这是将您的类注册到GTSAM框架的方式。`internal::Manifold`辅助类型捆绑了`ManifoldTraits`和`Testable`特征。
    ```cpp
    namespace gtsam {
    template <>
    struct traits<MyManifold> : public internal::Manifold<MyManifold> {};

    template <>
    struct traits<const MyManifold> : public internal::Manifold<MyManifold> {};
    }
    ```

### 3. `traits`特化的工作原理

通过提供`traits`特化，您并不是直接向类添加方法。相反，您使GTSAM的泛型全局函数能够与您的类型一起工作。当GTSAM算法调用如下函数时：

```cpp
gtsam::traits<MyManifold>::Local(origin, other);
```

编译器通过您的traits特化，将其翻译为您实现的成员函数调用：

```cpp
origin.localCoordinates(other);
```

这种设计使您的类保持干净，不包含框架特定的继承，同时允许与GTSAM的泛型编程模型完全集成。

### 4. 概念检查
为确保您的实现正确，请将以下宏添加到您的单元测试文件(testMyManifold.cpp)中。如果缺少任何Manifold要求，它将产生编译时错误。

```cpp
#include <gtsam/base/TestableAssertions.h>
#include <gtsam/base/Manifold.h>

// Your class include
#include <gtsam/geometry/MyManifold.h>

// This macro will fail to compile if MyManifold is not a valid Manifold.
GTSAM_CONCEPT_MANIFOLD_INST(MyManifold)

// ... your unit tests ...
```

这是验证您的类是否满足与GTSAM框架约定最有效的方法。

## MatrixLieGroup 矩阵李群
# 创建`MatrixLieGroup`

本指南说明如何创建`MatrixLieGroup`，它是一种具有矩阵表示的特定类型的`LieGroup`。**请先阅读[Manifold.md](Manifold.md)和[LieGroup.md](LieGroup.md)。**本文档仅关注矩阵李群的额外要求。

`MatrixLieGroup`具有`LieGroup`的所有性质，但其结构与一个底层的`N x N`矩阵相关联。这允许进行矩阵代数特有的额外操作。

### 1. 从LieGroup到MatrixLieGroup

要实现一个`MatrixLieGroup`，需要继承`gtsam::MatrixLieGroup<MyGroup, D, N>`，它又继承自`gtsam::LieGroup`。此继承免费提供更多功能。

### 2. 最低额外要求

您必须实现`LieGroup.md`指南中的所有内容，外加以下内容：

#### `MyGroup.h` - 头文件附加内容

1.  **类定义和继承**：
    *   将您的继承改为`public gtsam::MatrixLieGroup<MyGroup, D, N>`，其中`N`是矩阵的边长。

2.  **李代数Typedef**：
    *   `using LieAlgebra = gtsam::MatrixN;`：定义李代数(Lie algebra)元素的类型（即`N x N`矩阵）。

3.  **矩阵李群原语**：
    *   `const gtsam::MatrixN& matrix() const;`：底层矩阵数据成员的访问器。
    *   `static gtsam::MatrixN Hat(const gtsam::VectorD& xi);`：将一个`D`维切向量映射到其在李代数中的`N x N`矩阵表示。
    *   `static gtsam::VectorD Vee(const gtsam::MatrixN& X);`：`Hat`的逆操作，将一个`N x N`李代数矩阵映射回`D`维向量。

### 3. 免费获得的额外功能

通过继承`gtsam::MatrixLieGroup`，您将获得：

*   **默认群伴随方法**：`AdjointMap()`、`Adjoint(xi)`和`AdjointTranspose(x)`以泛型方式提供，包括后两者的Jacobian。
*   **默认李代数方法**：`adjointMap(xi)`、`adjoint(xi, y)`和`adjointTranspose(xi, y)`以泛型方式提供，包括后两者的Jacobian。
*   **`vec()`方法**：将群元素的`N x N`矩阵表示向量化为`(N*N) x 1`向量。

**性能说明：**`AdjointMap()`的泛型实现是正确的，但由于是通过`Hat`/`Vee`映射推导而来，可能会较慢。如果您的群存在`AdjointMap()`的闭式表达式，请考虑覆盖默认实现以获得更好的性能。

### 4. Traits和概念检查

最后，头文件中的traits特化必须更新以反映您的类现在是一个`MatrixLieGroup`。

1.  **GTSAM Traits特化**：
    *   这是最后的关键步骤。使用`internal::MatrixLieGroup`辅助类型特化`traits`结构体。
    ```cpp
    namespace gtsam {
    template <>
    struct traits<MyGroup> : public internal::MatrixLieGroup<MyGroup, N> {};

    template <>
    struct traits<const MyGroup> : public internal::MatrixLieGroup<MyGroup, N> {};
    }
    ```

2.  **概念检查**：
    *   更新单元测试文件中的宏以检查`MatrixLieGroup`概念。
    ```cpp
    #include <gtsam/base/TestableAssertions.h>
    #include <gtsam/base/MatrixLieGroup.h>

    // Your class include
    #include <gtsam/geometry/MyGroup.h>

    // This macro will fail to compile if MyGroup is not a valid MatrixLieGroup.
    GTSAM_CONCEPT_MATRIX_LIE_GROUP_INST(MyGroup)

    // ... your unit tests ...
    ```
这种模块化方法确保您的类提供了完全集成到GTSAM框架所需的所有组件。

## VectorSpace 向量空间
# 创建`VectorSpace`

本指南说明创建满足`VectorSpace`概念的类的要求。在GTSAM中，`VectorSpace`是一个特化的`AdditiveGroup`，添加了标量乘法和向量范数的概念。这是行为类似数学向量的类型应满足的概念。

**前提条件：**请阅读[Group.md](Group.md)，因为`VectorSpace`必须首先满足`AdditiveGroup`的所有要求。

---

### 1. 核心思想：可缩放的向量

`VectorSpace`直接建立在`AdditiveGroup`概念之上。它表示可以相加和相减的对象，但有一个关键补充：它们可以通过标量值（`double`）进行**缩放**。这允许诸如`0.5 * v`的操作，这在优化和线性代数中至关重要。

---

### 2. 最低实现要求

要成为`VectorSpace`，您的类必须首先实现`AdditiveGroup`和`Testable`所需的所有内容，外加一些新方法。

#### A `AdditiveGroup`要求（简要回顾）

您的类必须提供：
*   静态`Identity`方法（返回零向量）。
*   用于复合的`operator+`（向量加法）。
*   用于`Between`的二元`operator-`（向量减法）。
*   用于`Inverse`的一元`operator-`（向量取反）。

#### B 新的`VectorSpace`要求

除了群操作之外，您的类还必须提供：

*   **标量乘法**：一个`operator*`，以`double`作为*第一个*参数，以您的类类型作为第二个参数。
*   **维度**：一个实例方法`dim()`，以`int`或`size_t`返回向量的维度。固定大小和动态大小的向量空间都需要此方法。
*   **点积**：一个实例方法`dot`，计算与另一个类实例的点积。
*   **范数**：一个实例方法`norm`，计算向量的L2范数。

#### C `Testable`前提条件

与所有概念一样，您还必须提供：
*   `print`方法。
*   `equals`方法。

---

### 3. 将类链接到GTSAM（`traits`特化）

要将您的类注册为`VectorSpace`，请为您的类特化`gtsam::traits`结构体。此特化必须继承自`internal::VectorSpace<MyVectorSpace>`辅助类型。此辅助类型捆绑了`AdditiveGroup`和`Testable`特征，并添加了`vector_space_tag`。

您将在`gtsam`命名空间中添加一个继承自`internal::VectorSpace<MyVectorSpace>`的`traits<MyVectorSpace>`特化。

---

### 4. 对常见类型的内置支持

`VectorSpace.h`的一个关键特性是，它已为一些常见的C++和Eigen类型提供了完整的`VectorSpace`实现。**如果您使用以下任何一种类型，则无需编写自定义类或traits特化**：

*   **原始标量类型**：`double`和`float`自动被视为1D向量空间。
*   **固定大小Eigen矩阵和向量**：所有固定大小的Eigen类型，如`gtsam::Vector3`（`Eigen::Vector3d`）、`gtsam::Matrix2`（`Eigen::Matrix2d`）等，都被视为向量空间，其中维度是元素总数（例如，2x3矩阵的维度为6）。
*   **动态大小Eigen矩阵和向量**：所有动态大小的Eigen类型，包括`gtsam::Vector`（`Eigen::VectorXd`）和`Eigen::MatrixXd`，均完全支持。

这种内置支持意味着您可以在需要`VectorSpace`的GTSAM算法中直接使用Eigen类型，无需任何额外设置。

---

### 5. 概念检查

要验证您的自定义实现是否正确，请将`GTSAM_CONCEPT_VECTOR_SPACE_INST`宏添加到您的单元测试文件中。如果未满足`VectorSpace`（包括`AdditiveGroup`和`Testable`）的任何要求，这将导致编译时错误。

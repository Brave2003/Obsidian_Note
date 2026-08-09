# 表达式(Expressions)
## 动机
GTSAM是一个优化库，用于在一组未知变量上表示为因子图(factor graph)的目标函数。在连续情况下，变量通常是向量或流形(manifold)上的元素（如3D旋转流形）。因子计算需要最小化的向量值误差，通常只与少数未知量相连接。

在连续情况下，我们实现的主要优化方法是高斯-牛顿(Gauss-Newton)非线性优化或共轭梯度法的变体。假设有m个因子和n个未知量。对于任何一种优化方法，我们都需要计算整个因子图的稀疏Jacobian矩阵，它是一个有m个块行和n个块列的稀疏块矩阵。

稀疏Jacobian是逐个因子构建的，对应块行。一个典型的非线性最小二乘项为$|h(x)-z|^2$，其中$h(x)$是一个测量函数，我们需要能够将其线性化为
$$
h(x) \approx h(x_0+dx)+H(x_0)dx
$$
注意以上是针对向量未知量的，对于李群和流形变量，详见[doc/math.pdf](https://github.com/borglab/gtsam/blob/develop/doc/math.pdf)。

## 表达式
在许多情况下，可以使用GTSAM 4的表达式(Expression)来实现因子。表达式是Expression<T>类型的对象，主要有三种表达式风格：

- 常量，例如 `Expression<Point2> kExpr(Point2(3,4))`
- 未知量，例如 `Expression<Point3> pExpr(123)`，其中123是一个键(key)。
- 函数，例如 `Expression<double> sumExpr(h, kexpr, pExpr)`

后一种情况是包装一个二元测量函数`h`的示例。要能够包装`h`，它需要能够计算其局部导数，即必须具有以下签名：
```c++
double h(const Point2& a, const Point3& b,
         OptionalJacobian<1, 2> Ha, OptionalJacobian<1, 3> Hb)
```
在这种情况下，输出类型'T'是'double'，两个参数分别具有Point2和Point3类型，另外两个参数提供了一种在需要时计算函数Jacobian的方式。模板类型`OptionalJacobian<M,N>`的行为非常像`std::optional<Eigen::Matrix<double,M,N>>`。如果传入了实际的矩阵，函数应将其视为一个输出参数，在其中写入结果关于相应输入参数的Jacobian。*要写入的矩阵将在调用前分配好。*

表达式构造函数同时支持方法和不同参数数量的函数。注意表达式以输出类型T为模板参数，而不是以参数类型。然而，构造函数将通过检查函数f的签名来推断参数类型，并在此示例中期望另外两个参数，类型分别为Expression<Point2>和Expression<Point3>。

作为一个例子，以下是包装一元函数的构造函数声明：
```c++
template<typename A>
Expression(typename UnaryFunction<A>::type function,
    const Expression<A>& expression);
```
其中（在这种情况下）函数类型由以下定义：
```c++
template<class A1>
struct UnaryFunction {
typedef boost::function<
    T(const A1&, typename MakeOptionalJacobian<T, A1>::type)> type;
};
```
## 一些测量函数示例
一个简单一元函数的例子是`gtsam::norm3`，位于[Point3.cpp](https://github.com/borglab/gtsam/blob/develop/gtsam/geometry/Point3.cpp#L41)：
```c++
double norm3(const Point3 & p, OptionalJacobian<1, 3> H = {}) {
  double r = sqrt(p.x() * p.x() + p.y() * p.y() + p.z() * p.z());
  if (H) *H << p.x() / r, p.y() / r, p.z() / r;
  return r;
}
```
这里的关键新概念是OptionalJacobian，它类似于std::optional：如果它为true，您应该在其中写入函数的Jacobian。它作为一个固定大小的Eigen矩阵。

如上所述，表达式也支持二元函数、三元函数和方法。二元函数的一个例子是'Point3::cross'：

```c++
Point3 cross(const Point3 &p, const Point3 & q,
    OptionalJacobian<3, 3> H1 = {}, OptionalJacobian<3, 3> H2 = {}) {
  if (H1) *H1 << skewSymmetric(-q.x(), -q.y(), -q.z());
  if (H2) *H2 << skewSymmetric(p.x(), p.y(), p.z());
  return Point3(p.y() * q.z() - p.z() * q.y(), p.z() * q.x() - p.x() * q.z(),  p.x() * q.y() - p.y() * q.x());
}
```
使用cross的例子：
```c++
using namespace gtsam;
Matrix3 H1, H2;
Point3 p(1,2,3), q(4,5,6), r = cross(p,q,H1,H2);
```
## 使用表达式进行推断
表达式的使用方式是为我们要优化的未知变量创建未知表达式：
```c++
Expression<Point3> x(49;x49;,1);
auto h = Expression<Point3>(& norm3, x);
```
为了方便地使用表达式创建因子，我们提供了一个新的因子图类型`ExpressionFactorGraph`，它只是一个`NonlinearFactorGraph`，带有一个额外的方法addExpressionFactor(h, z, n)，该方法接受一个测量表达式h、一个实际测量值z和一个测量噪声模型R。通过这个，我们可以向`NonlinearFactorGraph`添加一个GTSAM非线性因子$|h(x)-z|^2$：
```c++
graph.addExpressionFactor(h, z, R)
```
在上面的例子中，未知量可以通过`gtsam::Symbol(49;x49;,1)`检索，它计算出一个uint64标识符。

## 组合表达式
然而，表达式背后真正的酷炫之处在于，您可以将它们组合成表达式树，只要叶子节点知道如何计算自己的导数：
```c++
Expression<Point3> x1(49;x49;1), x2(49;x49;,2);
auto h = Expression<Point3>(& cross, x1, x2);
auto g = Expression<Point3>(& norm3, h);
```
因为我们将Point3_ typedef为Expression<Point3>，我们可以非常简洁地写成：
```c++
auto g = Point3_(& norm3, Point3_(& cross, x1(49;x49;1), x2(49;x49;,2)));
```
## PoseSLAM示例
使用表达式，可以轻松快速地创建对应于PoseSLAM问题的因子图，在这种问题中，我们唯一的测量是一系列未知2D或3D位姿之间的相对位姿。以下来自[Pose2SLAMExampleExpressions.cpp](https://github.com/borglab/gtsam/blob/develop/examples/Pose2SLAMExampleExpressions.cpp)的代码片段用于创建一个简单的Pose2示例（机器人在平面上移动）：
```c++
1  ExpressionFactorGraph graph;
2  Expression<Pose2> x1(1), x2(2), x3(3), x4(4), x5(5);
3  graph.addExpressionFactor(x1, Pose2(0, 0, 0), priorNoise);
4  graph.addExpressionFactor(between(x1,x2), Pose2(2, 0, 0     ), model);
5  graph.addExpressionFactor(between(x2,x3), Pose2(2, 0, M_PI_2), model);
6  graph.addExpressionFactor(between(x3,x4), Pose2(2, 0, M_PI_2), model);
7  graph.addExpressionFactor(between(x4,x5), Pose2(2, 0, M_PI_2), model);
8  graph.addExpressionFactor(between(x5,x2), Pose2(2, 0, M_PI_2), model);
```
这是每一步的解释：
- 在第1行，我们创建一个空的因子图。
- 在第2行，我们创建5个未知位姿，类型为`Expression<Pose2>`，键从1到5。这些是我们将要优化的变量。
- 第3行创建一个简单因子，对`x1`（第一个参数）给出先验，即它位于原点`Pose2(0, 0, 0)`（第二个参数），具有由`priorNoise`（第三个参数）给出的特定概率密度。
- 第4-7行为里程计约束添加因子，即机器人连续位姿之间的运动。函数`between(t1,t2)`实现在[nonlinear/expressions.h](https://github.com/borglab/gtsam/blob/develop/gtsam/nonlinear/expressions.h)中，等效于调用构造函数Expression<T>(traits<T>::Between, t1, t2)。
- 最后，第8行在位姿x2和x5之间创建一个回环闭合(loop closure)约束。

另一个使用表达式的很好示例是[SFMExampleExpressions.cpp](https://github.com/borglab/gtsam/blob/develop/examples/SFMExampleExpressions.cpp)。

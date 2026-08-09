# Basis 基函数模块

`basis`模块提供了将连续函数表示为基函数(basis function)的线性组合或在插值点处的值的工具。它对于平滑函数逼近、轨迹建模以及构建在特定点约束函数（或其导数）的因子非常有用。

在高层次上，您选择一个基（Fourier或Chebyshev），决定是要基于系数的表示还是伪谱(pseudo-spectral)表示（在Chebyshev点处的值），然后使用提供的因子或拟合工具在GTSAM中求解参数。

## 入门指南

- **基于系数的基**：`Chebyshev1Basis`、`Chebyshev2Basis`和`FourierBasis`将参数视为基函数上的系数。
- **伪谱基**：`Chebyshev2`将参数视为Chebyshev点处的值，并使用重心插值(barycentric interpolation)。
- **因子**：一系列一元因子在特定点约束函数值或导数，包括向量值和流形值变体。
- **拟合**：`FitBasis`从样本执行最小二乘回归。

## 核心概念

- [Basis](doc/Basis.ipynb)：CRTP基类，提供求值和导数函子、Jacobian和通用辅助函数。

## 多项式基

- [Chebyshev1Basis](doc/Chebyshev1Basis.ipynb)：第一类Chebyshev基$T_n(x)$，区间$[-1,1]$（基于系数）。
- [Chebyshev2Basis](doc/Chebyshev2Basis.ipynb)：第二类Chebyshev基$U_n(x)$，区间$[-1,1]$（基于系数）。
- [FourierBasis](doc/FourierBasis.ipynb)：实Fourier级数基，用于周期函数。

## 伪谱基

- [Chebyshev2](doc/Chebyshev2.ipynb)：Chebyshev点，重心插值，微分/积分矩阵和求积权重。

## 基函数求值因子

这些因子将基参数与值或导数的测量联系起来。

- [EvaluationFactor](doc/EvaluationFactor.ipynb)：在一点处的标量值。
- [VectorEvaluationFactor](doc/VectorEvaluationFactor.ipynb)：在一点处的向量值。
- [VectorComponentFactor](doc/VectorComponentFactor.ipynb)：向量值的单个分量。
- [ManifoldEvaluationFactor](doc/ManifoldEvaluationFactor.ipynb)：流形值测量（例如`Rot3`、`Pose3`）。

## 导数约束因子

- [DerivativeFactor](doc/DerivativeFactor.ipynb)：在一点处的标量导数。
- [VectorDerivativeFactor](doc/VectorDerivativeFactor.ipynb)：在一点处的向量导数。
- [ComponentDerivativeFactor](doc/ComponentDerivativeFactor.ipynb)：向量导数的单个分量。

## 从数据拟合

- [FitBasis](doc/FitBasis.ipynb)：从样本构建最小二乘问题并求解基参数。

## Basis 基函数
# Basis

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/Basis.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`Basis`类是一个CRTP基类，定义了将函数表示为基函数线性组合的通用工具。派生类提供`CalculateWeights`和`DerivativeWeights`，而`Basis`提供用于求值、向量值求值和导数计算的函子和辅助函数，包括关于参数的Jacobian。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `WeightMatrix(N, X)`和`WeightMatrix(N, X, a, b)`构建堆叠的权重矩阵。
- `EvaluationFunctor`在`x`处求值标量函数。
- `VectorEvaluationFunctor`从参数矩阵求值向量值函数。
- `VectorComponentFunctor`求值向量值函数的单个分量。
- `ManifoldEvaluationFunctor`通过局部坐标求值流形值函数。
- `DerivativeFunctor`、`VectorDerivativeFunctor`和`ComponentDerivativeFunctor`计算导数。
- `kroneckerProductIdentity`为向量值情况构建高效的块Jacobian。

## 派生类

- `Chebyshev1Basis`（第一类Chebyshev多项式）
- `Chebyshev2Basis`（第二类Chebyshev多项式）
- `FourierBasis`（实Fourier级数）
- `Chebyshev2`（伪谱Chebyshev点）

## 使用示例

此示例为Chebyshev基构建权重矩阵并在一点处求值Fourier权重。

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

N = 5
X = np.linspace(-1.0, 1.0, 5)
W = gtsam.Chebyshev2.WeightMatrix(N, X)
print("Chebyshev2 WeightMatrix shape:", np.asarray(W).shape)
print("First row:", np.asarray(W)[0])

x = 0.3
weights = gtsam.FourierBasis.CalculateWeights(N, x)
print("FourierBasis weights at x=0.3:", np.asarray(weights).ravel())
```

## 源码
- [Basis.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/Basis.h)

## Chebyshev1Basis
# Chebyshev1Basis

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/Chebyshev1Basis.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`Chebyshev1Basis`提供区间$[-1, 1]$上的第一类Chebyshev多项式基$T_n(x)$。参数是基函数的系数，使其成为用于平滑函数逼近的经典正交多项式展开。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `CalculateWeights(N, x, a=-1, b=1)`返回$1 \times N$的基权重。
- `DerivativeWeights(N, x, a=-1, b=1)`返回导数的权重。
- `WeightMatrix(N, X)`为样本点向量堆叠权重。

## 使用示例

在一点处求值第一类Chebyshev权重，并为多个样本点构建权重矩阵。

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

N = 6
x = 0.25
weights = gtsam.Chebyshev1Basis.CalculateWeights(N, x)
print("Weights at x=0.25:", np.asarray(weights).ravel())

X = np.linspace(-1.0, 1.0, 5)
W = gtsam.Chebyshev1Basis.WeightMatrix(N, X)
print("WeightMatrix shape:", np.asarray(W).shape)
print("Row for x=0:", np.asarray(W)[2])
```

## 源码
- [Chebyshev.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/Chebyshev.h)

## Chebyshev2
# Chebyshev2

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/Chebyshev2.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`Chebyshev2`实现了第二类Chebyshev点上的伪谱参数化。参数是Chebyshev点处的函数值而非系数，求值使用重心插值。该类还提供微分矩阵和用于谱微积分的精确矩形积分矩阵。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `Point(N, j[, a, b])`和`Points(N[, a, b])`返回Chebyshev点。
- `CalculateWeights(N, x[, a, b])`返回重心插值权重。
- `DerivativeWeights(N, x[, a, b])`返回导数权重。
- `DifferentiationMatrix(N[, a, b])`对N个节点处的值进行微分。
- `IntegrationMatrix(N[, a, b])`将N个导数值映射到N+1个精确反导数值，F(a)=0。
- `IntegrationWeights(N[, a, b])`和`DoubleIntegrationWeights(N[, a, b])`提供求积权重。

## 使用示例

此示例检查Chebyshev点、插值权重、微分矩阵和精确积分矩阵。

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

N = 8
points = gtsam.Chebyshev2.Points(N)
print("Chebyshev points:", np.asarray(points).ravel())

x = 0.2
weights = gtsam.Chebyshev2.CalculateWeights(N, x)
print("Interpolation weights at x=0.2:", np.asarray(weights).ravel())

D = gtsam.Chebyshev2.DifferentiationMatrix(N)
print("D shape:", np.asarray(D).shape)
print("First row of D:", np.asarray(D)[0])

P = gtsam.Chebyshev2.IntegrationMatrix(N)
integration_weights = gtsam.Chebyshev2.IntegrationWeights(N)
print("P shape:", np.asarray(P).shape)
print("P final row equals weights:", np.allclose(np.asarray(P)[-1], integration_weights))
```

## 源码
- [Chebyshev2.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/Chebyshev2.h)

## Chebyshev2Basis
# Chebyshev2Basis

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/Chebyshev2Basis.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`Chebyshev2Basis`提供$[-1, 1]$上的第二类Chebyshev多项式基$U_n(x)$。它与第一类多项式的导数相关，当直接在$U_n(x)$多项式的基中表达函数时很有用。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `CalculateWeights(N, x, a=-1, b=1)`返回基权重。
- `DerivativeWeights(N, x, a=-1, b=1)`返回导数权重。
- `WeightMatrix(N, X)`为样本点向量堆叠权重。

## 使用示例

求值第二类Chebyshev权重并为多个样本点构建权重矩阵。

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

N = 6
x = -0.3
weights = gtsam.Chebyshev2Basis.CalculateWeights(N, x)
print("Weights at x=-0.3:", np.asarray(weights).ravel())

X = np.linspace(-1.0, 1.0, 5)
W = gtsam.Chebyshev2Basis.WeightMatrix(N, X)
print("WeightMatrix shape:", np.asarray(W).shape)
print("Row for x=0:", np.asarray(W)[2])
```

## 源码
- [Chebyshev.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/Chebyshev.h)

## ComponentDerivativeFactor
# ComponentDerivativeFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/ComponentDerivativeFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`ComponentDerivativeFactor`是一个一元因子，在给定点处约束向量值基函数单个分量的导数。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `ComponentDerivativeFactor(key, z, model, P, N, i, x)`约束分量`i`的$\mathbf{f}'(x)$。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例
```cpp
auto model = gtsam::noiseModel::Isotropic::Sigma(1, 0.05);
size_t P = 3, N = 6, i = 2;
double x = 0.2;
double z = 0.0;
gtsam::ComponentDerivativeFactor<gtsam::Chebyshev2> factor(key, z, model, P, N, i, x);
```

## Python使用示例

```python
import gtsam

model = gtsam.noiseModel.Isotropic.Sigma(1, 0.05)
key = gtsam.symbol("c", 0)
factor = gtsam.ComponentDerivativeFactorChebyshev2(key, 0.0, model, 3, 6, 2, 0.2)
print(factor)
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## DerivativeFactor
# DerivativeFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/DerivativeFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`DerivativeFactor`是一个一元因子，在给定点处强制基函数导数的标量测量。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `DerivativeFactor(key, z, model, N, x)`将$f'(x)$约束为测量值`z`。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例

```cpp
auto model = gtsam::noiseModel::Isotropic::Sigma(1, 0.05);
size_t N = 8;
double x = -0.3;
double z = 0.0;
gtsam::DerivativeFactor<gtsam::Chebyshev2> factor(key, z, model, N, x);

```

## Python使用示例

```python
import gtsam

model = gtsam.noiseModel.Isotropic.Sigma(1, 0.05)
key = gtsam.symbol('c', 0)
factor = gtsam.DerivativeFactorChebyshev2(key, 0.0, model, 8, -0.3)
print(factor)
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## EvaluationFactor
# EvaluationFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/EvaluationFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`EvaluationFactor`是一个一元因子，在给定点处强制基函数的标量测量。它通常与伪谱基一起使用来直接约束函数值。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `EvaluationFactor(key, z, model, N, x)`将$f(x)$约束为测量值`z`。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例

```cpp
auto model = gtsam::noiseModel::Isotropic::Sigma(1, 0.05);
size_t N = 8;
double x = 0.25;
double z = 1.2;
gtsam::EvaluationFactor<gtsam::Chebyshev2> factor(key, z, model, N, x);
```

## Python使用示例

```python
import gtsam

key = gtsam.symbol('c', 0)
model = gtsam.noiseModel.Isotropic.Sigma(1, 0.05)
factor = gtsam.EvaluationFactorChebyshev2(key, 1.2, model, 8, 0.25)
print(factor)
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## FitBasis
# FitBasis

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/FitBasis.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`FitBasis`执行最小二乘回归，将基函数拟合到样本数据。它从样本构建因子图，并在高斯噪声模型下求解最能解释数据的基参数。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `FitBasis(sequence, model, N)`构造并求解最小二乘问题。
- `parameters()`返回拟合的参数向量。
- `NonlinearGraph(...)`和`LinearGraph(...)`暴露中间图。

## C++使用示例

```cpp
std::map<double, double> samples = {{0.0, 1.0}, {0.5, 0.2}, {1.0, -0.1}};
auto model = gtsam::noiseModel::Isotropic::Sigma(1, 0.1);
size_t N = 5;
gtsam::FitBasis<gtsam::Chebyshev2> fit(samples, model, N);
gtsam::Vector params = fit.parameters();

```

## Python示例

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

sequence = {0.0: 1.0, 0.5: 0.2, 1.0: -0.1}
model = gtsam.noiseModel.Isotropic.Sigma(1, 0.1)
fit = gtsam.FitBasisFourierBasis(sequence, model, 3)
params = fit.parameters()
print("FitBasisFourierBasis parameters:", params)
```

## 源码
- [FitBasis.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/FitBasis.h)

## FourierBasis
# FourierBasis

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/FourierBasis.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`FourierBasis`提供用于周期函数的实Fourier级数基。基按$[1, \cos(x), \sin(x), \cos(2x), \sin(2x), \ldots]$排序，便于截断Fourier展开及其导数。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `CalculateWeights(N, x)`返回实Fourier基权重。
- `DifferentiationMatrix(N)`返回导数的线性算子。
- `DerivativeWeights(N, x)`返回在`x`处求值导数的权重。

## 使用示例

求值实Fourier基、其导数权重和微分矩阵的一个切片。

```python
import numpy as np
import gtsam

np.set_printoptions(precision=3, suppress=True)

N = 7
x = 1.2
weights = gtsam.FourierBasis.CalculateWeights(N, x)
dweights = gtsam.FourierBasis.DerivativeWeights(N, x)
D = gtsam.FourierBasis.DifferentiationMatrix(N)

print("Weights:", np.asarray(weights).ravel())
print("Derivative weights:", np.asarray(dweights).ravel())
print("D[1:3, 1:3]:", np.asarray(D)[1:3, 1:3])
```

## 源码
- [Fourier.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/Fourier.h)

## ManifoldEvaluationFactor
# ManifoldEvaluationFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/ManifoldEvaluationFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`ManifoldEvaluationFactor`是一个用于流形值测量（如`Rot2`、`Rot3`、`Pose2`或`Pose3`）的一元因子。它求值向量值基函数，并在流形上的局部坐标中比较结果。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `ManifoldEvaluationFactor(key, z, model, N, x)`在`x`处约束流形值。
- 可选的区间参数`a, b`缩放基的定义域。

## C++示例

```cpp
auto model = gtsam::noiseModel::Isotropic::Sigma(3, 0.05);
size_t N = 6;
double x = 0.1;
gtsam::Rot3 z = gtsam::Rot3::RzRyRx(0.1, 0.2, 0.3);
gtsam::ManifoldEvaluationFactor<gtsam::Chebyshev2, gtsam::Rot3> factor(key, z, model, N, x);
```

## Python示例

```python
import numpy as np
import gtsam

names = [n for n in dir(gtsam) if n.startswith("ManifoldEvaluationFactor")]
print("Available ManifoldEvaluationFactor wrappers:")
for name in names:
    print("  ", name)

selected = None
kind = None
for preferred in ["Rot2", "Pose2", "Rot3", "Pose3"]:
    for name in names:
        if preferred in name:
            selected = name
            kind = preferred
            break
    if selected is None:
        print("No ManifoldEvaluationFactor wrapper found; skipping execution.")
    else:
        cls = getattr(gtsam, selected)
        if kind == "Rot2":
            z = gtsam.Rot2(0.3)
            dim = 1
        elif kind == "Rot3":
            z = gtsam.Rot3.RzRyRx(0.1, 0.2, 0.3)
            dim = 3
        elif kind == "Pose2":
            z = gtsam.Pose2(1.0, 0.0, 0.1)
            dim = 3
        else:
            z = gtsam.Pose3(gtsam.Rot3.RzRyRx(0.1, 0.2, 0.3), np.array([1.0, 0.0, 0.0]))
            dim = 6

        model = gtsam.noiseModel.Isotropic.Sigma(dim, 0.1)
        key = gtsam.symbol('x', 0)
        factor = cls(key, z, model, 6, 0.1)
        print(f"Created {selected} with dim {factor.dim()}")
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## VectorComponentFactor
# VectorComponentFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/VectorComponentFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`VectorComponentFactor`是一个一元因子，在给定点处约束向量值基函数的单个分量。当只有一个分量被观测到时，这很有用。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `VectorComponentFactor(key, z, model, P, N, i, x)`约束分量`i`。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例

```cpp
auto model = gtsam::noiseModel::Isotropic::Sigma(1, 0.1);
size_t P = 3, N = 6, i = 1;
double x = 0.5;
double z = -0.2;
gtsam::VectorComponentFactor<gtsam::Chebyshev2> factor(key, z, model, P, N, i, x);
```

## Python示例

```python
import gtsam

candidates = [n for n in dir(gtsam) if n.startswith("VectorComponentFactor")]
print("VectorComponentFactor wrappers:")
for name in candidates:
    print("  ", name)

cls = None
selected = None
for name in candidates:
    if hasattr(gtsam, name):
        cls = getattr(gtsam, name)
        selected = name

    if cls is None:
        print("No VectorComponentFactor wrapper found; skipping execution.")
    else:
        model = gtsam.noiseModel.Isotropic.Sigma(1, 0.1)
        key = gtsam.symbol('c', 0)
        factor = cls(key, -0.2, model, 3, 6, 1, 0.5)
        print(f"Created {selected} with dim {factor.dim()}")
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## VectorDerivativeFactor
# VectorDerivativeFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/VectorDerivativeFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`VectorDerivativeFactor`是一个一元因子，在给定点处强制向量值基函数导数的向量测量。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `VectorDerivativeFactor(key, z, model, M, N, x)`约束$\mathbf{f}'(x)$。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例

```cpp
gtsam::Vector z = (gtsam::Vector(2) << 0.1, -0.1).finished();
auto model = gtsam::noiseModel::Isotropic::Sigma(2, 0.05);
size_t M = 2, N = 6;
double x = 0.0;
gtsam::VectorDerivativeFactor<gtsam::Chebyshev2> factor(key, z, model, M, N, x);
```

## Python示例

```python
import numpy as np
import gtsam

candidates = [n for n in dir(gtsam) if n.startswith("VectorDerivativeFactor")]
print("VectorDerivativeFactor wrappers:")
for name in candidates:
    print("  ", name)

z = np.array([0.1, -0.1])
model = gtsam.noiseModel.Isotropic.Sigma(2, 0.05)
key = gtsam.symbol('c', 0)
factor = gtsam.VectorDerivativeFactorChebyshev2(key, z, model, 2, 6, 0.0)
print("factor:\n", factor)
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

## VectorEvaluationFactor
# VectorEvaluationFactor

<a href="https://colab.research.google.com/github/borglab/gtsam/blob/develop/gtsam/basis/doc/VectorEvaluationFactor.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

## 概述

`VectorEvaluationFactor`是一个一元因子，用于从基参数矩阵求值的向量值测量。它强制$\mathbf{f}(x)$在给定点处匹配测量向量。

GTSAM Copyright 2010-2022, Georgia Tech Research Corporation,
Atlanta, Georgia 30332-0415
All Rights Reserved

Authors: Frank Dellaert, et al. (see THANKS for the full author list)

See LICENSE for the license information

```python
# Install GTSAM from pip if running in Google Colab
try:
    import google.colab
    %pip install --quiet gtsam-develop
except ImportError:
    pass  # Not in Colab
```

## 关键功能/API

- `VectorEvaluationFactor(key, z, model, M, N, x)`约束完整向量。
- 可选的区间参数`a, b`缩放基的定义域。

## C++使用示例

```cpp
gtsam::Vector z = (gtsam::Vector(3) << 1.0, 0.0, -0.5).finished();
auto model = gtsam::noiseModel::Isotropic::Sigma(3, 0.1);
size_t M = 3, N = 6;
double x = 0.0;
gtsam::VectorEvaluationFactor<gtsam::Chebyshev2> factor(key, z, model, M, N, x);
```

## Python示例

```python
import numpy as np
import gtsam

candidates = [n for n in dir(gtsam) if n.startswith("VectorEvaluationFactor")]
print("VectorEvaluationFactor wrappers:")
for name in candidates:
    print("  ", name)

cls = None
selected = None
for name in [
    "VectorEvaluationFactorChebyshev2",
    "VectorEvaluationFactorChebyshev2Basis",
    "VectorEvaluationFactorChebyshev1Basis",
    "VectorEvaluationFactorFourierBasis",
]:
    if hasattr(gtsam, name):
        cls = getattr(gtsam, name)
        selected = name

    if cls is None:
        print("No VectorEvaluationFactor wrapper found; skipping execution.")
    else:
        z = np.array([1.0, 0.0, -0.5])
        model = gtsam.noiseModel.Isotropic.Sigma(3, 0.1)
        key = gtsam.symbol('c', 0)
        factor = cls(key, z, model, 3, 6, 0.0)
        print(f"Created {selected} with dim {factor.dim()}")
```

## 源码
- [BasisFactors.h](https://github.com/borglab/gtsam/blob/develop/gtsam/basis/BasisFactors.h)

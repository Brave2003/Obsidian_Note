# SLAM算法工程师面试题汇总

> 原文来源：[知乎专栏 - SLAM面试题汇总](https://zhuanlan.zhihu.com/p/27904353682)
>
> SLAM算法岗位一般考察三个部分：**SLAM相关算法理论知识**、**C++语言与ROS**、**算法题（含LeetCode）**。

---

## 使用说明与面试回答方法

这份扩展版不是为了让你逐字背诵，而是帮助你形成“**定义 → 原理 → 数学模型 → 工程实现 → 优缺点 → 失败场景 → 项目经验**”的回答结构。

面试中建议采用下面的四层回答法：

1. **先给一句话结论**：说明算法属于哪一类，解决什么问题。
2. **再讲核心流程**：输入、状态、残差/观测、求解、输出。
3. **补充工程取舍**：计算量、实时性、初始化、退化、异常值。
4. **结合项目闭环**：你遇到什么问题，如何定位，修改后指标提升多少。

文中标记：

- **高频**：基础岗位经常直接问。
- **追问**：面试官可能顺着答案继续问。
- **易错点**：常见但不严谨的说法。
- **项目回答模板**：可替换成自己的数据、模块和结果。

> 推荐复习顺序：视觉/VIO岗位优先看第二、三、四、八、十二章；激光岗位优先看第二、三、五、六、八、十二章；工程岗位还需重点看第九、十、十一、十三章。


## 一、基础数学部分

### 1. 请谈谈对流形的理解？

**解答：** 流形（Manifold）是一个局部同胚于欧几里得空间的拓扑空间。通俗地说，流形在局部看起来像 $\mathbb{R}^n$，但全局结构可能更复杂（如球面、环面）。

在SLAM中，流形的核心意义在于：旋转矩阵 $SO(3)$、刚体变换 $SE(3)$ 都是流形，它们不是向量空间（不能直接相加），但在局部切空间（李代数）中可以自由进行线性运算，然后通过指数映射回到流形上。

**关键性质：** 流形上的优化不能直接用 $x + \Delta x$ 的欧式更新，而需要用 $x \oplus \Delta x = \exp(\Delta x^\wedge) \cdot x$ 的流形更新（retraction）。

流形就是局部平直、全局弯曲的空间。SLAM 中的旋转、刚体位姿属于流形，无法直接欧式加减。我们借助李群李代数，把流形上微小扰动放到切空间（李代数）做线性计算，再通过指数映射更新流形上的状态，保证优化过程中参数始终合法。

**参考：**
- [A micro Lie theory for state estimation in robotics (Sola, 2018)](https://arxiv.org/abs/1812.01537)
- [SLAM十四讲 第4讲：李群与李代数](https://github.com/gaoxiang12/slambook2)

---

### 2. 什么是李群？什么是李代数？

**解答：**

**李群（Lie Group）** 是一种同时具有**群结构**和**光滑流形结构**的数学对象，且群运算（乘法和求逆）是光滑映射：
- **群结构**：满足封闭性、结合律、有单位元、有逆元
- **光滑流形结构**：局部同胚于欧式空间，可以在其上做微积分

典型例子：$SO(3)$（三维旋转群）、$SE(3)$（三维刚体变换群）。

**李代数（Lie Algebra）** 是与李群相关联的一个线性空间，具体定义为李群在**单位元处的切空间**，并配备**李括号** $[\cdot, \cdot]$ 运算：
- 描述了李群在单位元附近的局部性质
- 是一个向量空间，可以自由进行加减运算
- 通过指数映射 $\exp(\cdot)$ 与李群连接

例如 $so(3)$ 是 $SO(3)$ 在 $I$ 处的切空间，元素为 $\phi \in \mathbb{R}^3$（旋转向量），李括号为 $[\phi_1, \phi_2] = \phi_1^\wedge \phi_2^\wedge - \phi_2^\wedge \phi_1^\wedge$。

**参考：**
- [State Estimation for Robotics - Barfoot, 第7章](http://asrl.utias.utoronto.ca/~tdb/bib/barfoot_ser17.pdf)
- [SLAM十四讲 第4讲](https://github.com/gaoxiang12/slambook2)

---

### 3. 请谈谈李群和李代数的关系？

**解答：**

从数学定义上来说：
1. 李代数是李群**单位元处的切空间**并配备李括号运算，每个李群对应一个李代数
2. 李代数描述了李群在单位元附近的**局部结构**
3. 李代数通过**指数映射**生成李群：$\exp: \mathfrak{g} \to G$
4. 李群通过**对数映射**回到李代数：$\log: G \to \mathfrak{g}$

具体对应关系：
| 李群 | 李代数 | 含义 |
|------|--------|------|
| $SO(3)$ | $so(3) \cong \mathbb{R}^3$ | 三维旋转 |
| $SE(3)$ | $se(3) \cong \mathbb{R}^6$ | 三维刚体变换 |
| $SO(2)$ | $so(2) \cong \mathbb{R}$ | 二维旋转 |
| $SE(2)$ | $se(2) \cong \mathbb{R}^3$ | 二维刚体变换 |

在SLAM中的应用：在位姿图优化中，用 $se(3)$ 中的增量 $\Delta \xi$ 来更新 $SE(3)$ 中的位姿 $T$：$T \leftarrow \exp(\Delta \xi^\wedge) \cdot T$。

**参考：**
- [A micro Lie theory for state estimation in robotics](https://arxiv.org/abs/1812.01537)
- [Sophus库 - C++ Lie群实现](https://github.com/strasdat/Sophus)

---

### 4. 雅可比矩阵、扰动模型和BCH近似公式的理解

**解答：**

**雅可比矩阵（Jacobian）：** 在SLAM优化中，Jacobian表示残差对优化变量的偏导数 $\frac{\partial e}{\partial x}$。在非线性最小二乘中，Jacobian用于构建正规方程 $J^T J \Delta x = -J^T e$。李群上的Jacobian需要特殊处理，因为优化变量在流形上。

**扰动模型（Perturbation Model）：** 对李群元素施加微小扰动 $\delta \xi$（在李代数上）来求导，而非直接对李代数元素求导。分为：
- **左扰动**：$T' = \exp(\delta \xi^\wedge) \cdot T$，$\frac{\partial T'}{\partial \delta \xi} \big|_{\delta \xi = 0}$
- **右扰动**：$T' = T \cdot \exp(\delta \xi^\wedge)$，$\frac{\partial T'}{\partial \delta \xi} \big|_{\delta \xi = 0}$

左扰动和右扰动都很常见，不能简单说哪一种必然更好。关键是明确误差定义在哪个坐标系，并保证状态更新、伴随矩阵、Jacobian和协方差传播使用同一套约定。

**BCH近似公式（Baker-Campbell-Hausdorff）：** 当两个李代数元素做指数映射后的乘积需要用一个李代数表示时：
$$\ln(\exp(A^\wedge)\exp(B^\wedge))^\vee \approx \begin{cases} A + B + \frac{1}{2}[A, B] + \cdots & \text{(BCH公式)} \\ J_l^{-1}(A) B + A & \text{(左乘近似)} \\ J_r^{-1}(B) A + B & \text{(右乘近似)} \end{cases}$$

其中 $J_l$ 和 $J_r$ 是左/右Jacobian。BCH公式告诉我们在流形上"加法"不是简单的向量加法。

**参考：**
- [SLAM十四讲 第4讲：李群与李代数](https://github.com/gaoxiang12/slambook2)
- [State Estimation for Robotics, Barfoot - 第7章](http://asrl.utias.utoronto.ca/~tdb/bib/barfoot_ser17.pdf)

---


## 二、概率统计与非线性优化基础

### 1. 最大似然估计MLE、最大后验估计MAP和最小二乘是什么关系？【高频】

**一句话回答：** 最小二乘可以看作高斯噪声假设下的最大似然估计；加入状态先验后，就变成最大后验估计。

设观测模型为：

$$z_i=h_i(x)+n_i,\qquad n_i\sim\mathcal N(0,\Sigma_i)$$

似然为：

$$p(z_i|x)\propto\exp\left(-\frac12\|z_i-h_i(x)\|_{\Sigma_i^{-1}}^2\right)$$

对全部独立观测求最大似然，等价于最小化负对数似然：

$$x^*_{\text{MLE}}=\arg\min_x\sum_i\|z_i-h_i(x)\|_{\Sigma_i^{-1}}^2$$

如果还有先验 $x\sim\mathcal N(\bar x,\Sigma_0)$，则：

$$x^*_{\text{MAP}}=\arg\min_x\left(\|x-\bar x\|_{\Sigma_0^{-1}}^2+\sum_i\|z_i-h_i(x)\|_{\Sigma_i^{-1}}^2\right)$$

**面试追问：为什么是信息矩阵加权？**

因为协方差越小代表观测越可信，对应信息矩阵 $\Omega=\Sigma^{-1}$ 越大，残差在目标函数中的惩罚越强。

**易错点：** MAP不等于“比MLE一定更准确”，先验错误时反而会引入偏差。

---

### 2. 梯度下降、高斯—牛顿和Levenberg–Marquardt有什么区别？【高频】

对于非线性最小二乘：

$$\min_x \frac12\|r(x)\|^2$$

在当前状态处线性化：

$$r(x+\delta x)\approx r(x)+J\delta x$$

**梯度下降：**

$$\delta x=-\alpha J^Tr$$

只使用一阶方向，单步便宜但收敛慢，步长难选。

**高斯—牛顿：**

$$J^TJ\delta x=-J^Tr$$

用 $J^TJ$ 近似真实Hessian。接近最优解且残差较小时收敛快，但初值差或Hessian病态时可能发散。

**LM：**

$$\left(J^TJ+\lambda D\right)\delta x=-J^Tr$$

通过阻尼在梯度下降和高斯—牛顿之间切换。步子有效就减小 $\lambda$，无效就增大 $\lambda$。

**工程回答：** VINS、BA、点云配准中通常使用GN或LM；实时系统会限制迭代次数，并结合鲁棒核、状态备份和步长接受策略。

---

### 3. 为什么不建议直接计算矩阵逆？

表达式 $x=A^{-1}b$ 在代码里通常不应写成 `A.inverse() * b`，而应使用分解求解：

- 对称正定：Cholesky/LLT；
- 对称半正定或可能不定：LDLT；
- 一般稠密矩阵：QR；
- 秩亏或需要最小范数解：SVD；
- 大规模稀疏系统：Sparse Cholesky、PCG等。

原因：显式求逆计算量更大、数值误差更明显，而且破坏稀疏结构。

**面试追问：正规方程为什么可能不稳定？**

因为 $H=J^TJ$ 的条件数近似为 $\kappa(H)=\kappa(J)^2$，会放大病态性。QR或平方根方法直接作用于Jacobian，通常更稳定。

---

### 4. Schur Complement在BA中解决什么问题？【高频】

BA状态可分为相机变量 $x_c$ 和路标变量 $x_l$：

$$
\begin{bmatrix}
H_{cc}&H_{cl}\\
H_{lc}&H_{ll}
\end{bmatrix}
\begin{bmatrix}\delta x_c\\\delta x_l\end{bmatrix}
=
\begin{bmatrix}b_c\\b_l\end{bmatrix}
$$

路标之间通常互不直接连接，所以 $H_{ll}$ 是块对角或易求逆的。消除路标：

$$\left(H_{cc}-H_{cl}H_{ll}^{-1}H_{lc}\right)\delta x_c
=b_c-H_{cl}H_{ll}^{-1}b_l$$

先求相机增量，再回代路标增量。

**优点：** 将大量三维点变量消掉，显著减少主求解系统维度。

**追问：MSCKF零空间投影和Schur补有关系吗？**

在线性高斯问题中，两者本质上都是消除特征变量，只是一个从Jacobian零空间角度表达，一个从正规方程消元角度表达。

---

### 5. 什么是边缘化？为什么滑动窗口需要边缘化？【高频】

滑动窗口不能无限增长，因此要删除旧状态，但又不能把旧观测携带的信息全部丢掉。边缘化通过变量消元，将旧变量的信息压缩为保留变量上的先验。

设变量为待保留 $x_r$ 和待删除 $x_m$，在线性系统上做Schur补后得到：

$$H'_r=H_{rr}-H_{rm}H_{mm}^{-1}H_{mr}$$

$$b'_r=b_r-H_{rm}H_{mm}^{-1}b_m$$

新的 $(H'_r,b'_r)$ 就是边缘化先验。

**主要问题：** 先验固定在生成它时的线性化点，后续不能像普通残差一样完全重新线性化，因此可能带来不一致性。

**工程措施：** FEJ、固定部分线性化点、控制窗口长度、避免过早边缘化弱观测状态、使用square-root marginalization。

---

### 6. 什么是Gauge Freedom和不可观方向？【高频】

如果对所有位姿同时施加某个全局变换，但所有相对观测残差不变，那么该方向无法由系统自身确定，称为规范自由度或不可观方向。

典型情况：

- 纯单目视觉：全局平移3维、全局旋转3维、尺度1维，共7维Sim(3)自由度；
- 双目/RGB-D视觉：尺度可观，但全局SE(3)仍需固定参考；
- 标准VIO：重力确定roll和pitch、尺度在充分激励下可观，剩余全局位置3维和全局yaw 1维不可观。

如果不固定规范自由度，Hessian会秩亏。常用方法是固定第一帧、增加软先验或使用零空间友好的求解方式。

---

### 7. 什么是可观性？如何判断系统退化？

可观性描述能否根据一段时间的输入和观测唯一恢复状态。线性系统可通过可观矩阵判断；非线性系统常通过局部线性化、Fisher信息矩阵或Hessian特征值分析。

工程中的退化信号：

- Hessian出现很小特征值；
- 某些状态协方差快速变大；
- 点云配准在某方向增量不稳定；
- 单目VIO纯旋转、匀速或低视差导致尺度/重力难初始化；
- 长走廊使沿走廊方向约束弱；
- 纯平面场景缺少部分旋转和平移约束。

处理方式：检测退化方向并降权、引入IMU/GNSS/轮速、延迟初始化、增加时间窗口或主动激励运动。

---

### 8. 协方差矩阵和信息矩阵如何理解？

协方差 $P$ 描述不确定性，信息矩阵 $\Lambda=P^{-1}$ 描述约束强度。

- $P$大：状态不确定；
- $\Lambda$大：信息强；
- 协方差形式适合滤波传播；
- 信息形式和平方根信息形式适合稀疏图优化与因子累加。

两个独立高斯先验融合时，信息可以直接相加：

$$\Lambda=\Lambda_1+\Lambda_2,\qquad
\eta=\eta_1+\eta_2$$

其中 $\eta=\Lambda\mu$。

---

### 9. 鲁棒核函数解决什么问题？Huber、Cauchy有什么区别？【高频】

普通最小二乘对大残差平方惩罚，少量错误匹配就可能主导优化。鲁棒核将代价改为 $\rho(s)$，其中 $s=r^T\Omega r$。

- **Huber**：小残差保持二次，大残差转为近似线性；稳定、常用；
- **Cauchy**：对大残差抑制更强，但可能使优化更非凸；
- **Tukey**：超过阈值后权重可变为零，抗离群强但依赖较好初值。

**易错点：** 鲁棒核不是异常点检测的替代品。错误回环、动态物体等最好先做几何验证和门控，再使用鲁棒核兜底。

---

### 10. 数值求导、解析求导、自动求导怎么选？

- **数值求导**：实现快，但精度受步长影响，速度慢，适合验证；
- **解析求导**：速度快、可控，但推导复杂，容易写错；
- **自动求导**：开发效率高，精度接近解析，Ceres常用；模板展开可能增加编译时间和运行开销。

工程做法：先用自动求导快速验证，再对热点残差实现解析Jacobian；通过有限差分做单元测试，比较相对误差。

---

### 11. SVD、特征值分解和QR在SLAM中分别用于什么？

- **SVD**：刚体配准、八点法、求零空间、判断秩、最小范数解；
- **特征值分解**：PCA法向估计、退化检测、协方差主方向；
- **QR**：最小二乘、边缘化、消除路标、避免构造 $J^TJ$；
- **Cholesky/LDLT**：求解对称线性系统；
- **Schur Complement**：BA中消除路标或边缘化变量。

面试时不要只背函数名，要说明矩阵性质决定求解器选择。

---

### 12. 左扰动和右扰动如何选择？

左扰动：

$$T'=\exp(\delta\xi^\wedge)T$$

右扰动：

$$T'=T\exp(\delta\xi^\wedge)$$

两者的误差分别常被解释在不同坐标系中，并通过伴随矩阵联系：

$$\delta\xi_{\text{left}}\approx\operatorname{Ad}_T\delta\xi_{\text{right}}$$

没有统一规定“左一定比右好”。必须保证以下内容一致：

- 状态更新；
- 残差定义；
- Jacobian；
- IMU误差模型；
- 协方差坐标系；
- Sophus/GTSAM/Ceres中的参数化约定。



## 三、多传感器融合

### 1. 请简述KF、EKF、ESKF的流程？三者的区别？

**解答：**

**卡尔曼滤波（KF - Kalman Filter）：**
- **适用**：线性系统 + 高斯噪声
- **预测**：$\hat{x}_{k|k-1} = F_k \hat{x}_{k-1}$，$P_{k|k-1} = F_k P_{k-1} F_k^T + Q_k$
- **更新**：$K_k = P_{k|k-1} H_k^T (H_k P_{k|k-1} H_k^T + R_k)^{-1}$，$\hat{x}_k = \hat{x}_{k|k-1} + K_k(z_k - H_k \hat{x}_{k|k-1})$，$P_k = (I - K_k H_k)P_{k|k-1}$

**扩展卡尔曼滤波（EKF - Extended Kalman Filter）：**
- **适用**：非线性系统（一阶线性化）
- **预测**：$\hat{x}_{k|k-1} = f(\hat{x}_{k-1}, u_k)$，$P_{k|k-1} = F_k P_{k-1} F_k^T + Q_k$，其中 $F_k = \frac{\partial f}{\partial x}\big|_{\hat{x}_{k-1}}$
- **更新**：同上，$H_k = \frac{\partial h}{\partial x}\big|_{\hat{x}_{k|k-1}}$

**误差状态卡尔曼滤波（ESKF - Error State Kalman Filter）：**
- **核心思想**：将状态分解为**名义状态（nominal state）+ 误差状态（error state）**
- 名义状态用非线性方程传播，误差状态用线性KF估计
- 每次KF更新后，将误差注入名义状态，然后误差状态归零（reset）
- **优势**：误差状态始终接近零，线性化精度高；姿态用四元数表示但误差用旋转向量，无过参数化

**三者区别：**
| 特性 | KF | EKF | ESKF |
|------|-----|------|------|
| 系统模型 | 线性 | 非线性 | 非线性 |
| 状态量 | 原始状态 | 原始状态 | 名义 + 误差 |
| 线性化点 | 无需 | 当前估计 | 在零误差附近建立误差模型，但Jacobian仍依赖当前名义状态 |
| 过参数化问题 | 无 | 可能有 | 无 |
| 姿态表示 | 不适用 | 四元数（有问题） | 名义四元数 + 误差旋转向量 |

**参考：**
- [Quaternion kinematics for the error-state Kalman filter (Sola, 2017)](https://arxiv.org/abs/1711.02508)
- [ESKF详细推导](https://github.com/gaoxiang12/slambook2)

---

### 2. 卡尔曼滤波中 Q 和 R 应该怎么调？

**解答：**

**Q（过程噪声协方差矩阵）** 描述系统模型的不确定性，即状态预测的误差大小。

**R（观测噪声协方差矩阵）** 描述观测数据的不确定性，即测量值的误差大小。

**调节原则：**
- 观测数据非常精确（如高精度LiDAR点云配准结果）→ **R 调小** → KF更信任观测
- 观测数据噪声较大（如低精度GPS）→ **R 调大** → KF更信任预测
- 系统模型非常准确（如高精度IMU、不打滑的轮式里程计）→ **Q 调小**
- 系统模型不准确（如imu bias漂移大）→ **Q 调大**

**工程实践方法：**
1. **Sage-Husa自适应滤波**：在线实时估计 Q 和 R
2. **保守初始化 + 场景调参**：先给较大Q/R，再根据典型场景（长廊、动态环境、转弯）测试调整
3. **基于残差匹配**：观测残差 $\nu_k = z_k - H \hat{x}_{k|k-1}$ 的协方差应与 $H P_{k|k-1} H^T + R$ 匹配，根据匹配程度调整

**参考：**
- [Kalman Filter Tuning - IEEE Paper](https://ieeexplore.ieee.org/document/4329175)
- [Sage-Husa Adaptive Kalman Filter](https://en.wikipedia.org/wiki/Kalman_filter#Adaptive_Kalman_filter)

---

### 3. ESKF中bias的reset方法一般有几种？有什么区别？

**解答：**

ESKF中误差状态估计出来后，需要将误差注入名义状态，然后将误差状态**归零（reset）**。bias的reset主要有以下方式：

**1. 加法reset（最常用）：**
- 名义bias直接加上误差bias：$b \leftarrow b + \delta b$
- 误差bias置零：$\delta b \leftarrow 0$
- 协方差不变（因为bias的误差模型是线性的）

**2. 不需要特殊处理：**
- IMU bias的状态转移是线性的（$\dot{b} = \eta_b$，随机游走），因此bias的误差状态的Jacobian就是单位阵
- reset时只需 $\delta b \leftarrow 0$，协方差不需要调整

**3. 姿态部分的reset（需要特殊处理）：**
- 姿态误差注入后需要使用reset Jacobian（重置雅可比）变换协方差
- 因为 $\delta \theta$ 被注入到四元数后归零，协方差矩阵需要相应调整：
  $$P \leftarrow J P J^T$$
  其中 $J$ 是reset操作的Jacobian

**区别：** 姿态误差reset需要调整协方差（因为这是非线性操作），bias误差reset是线性操作不需要调整协方差。

**参考：**
- [Quaternion kinematics for the error-state Kalman filter (Sola, 2017)](https://arxiv.org/abs/1711.02508)

---

### 4. 传感器的标定包括哪些？传感器的时间戳同步应该怎么做？

**解答：**

**传感器标定包括：**

**内参标定：**
- **相机内参**：焦距 $f_x, f_y$，主点 $c_x, c_y$，畸变系数 $k_1, k_2, p_1, p_2$
- **IMU内参**：加速度计/陀螺仪的scale、misalignment、bias
- **LiDAR内参**：各激光通道的俯仰角、水平角、距离偏移

**外参标定（传感器间变换）：**
- Camera-IMU外参 $T_{ci}$（VI传感器标定，如Kalibr）
- LiDAR-IMU外参 $T_{li}$（如LI-Init、Fast-LIO中的在线标定）
- LiDAR-Camera外参 $T_{lc}$（如ACSC方法）
- Multi-LiDAR之间的外参
- Vehicle-IMU外参

**时间戳同步方法：**

1. **硬件同步（硬触发）**：GPS PPS信号触发所有传感器，精度最高（微秒级）
2. **软件时间戳**：记录传感器数据到达处理器的时间
3. **时间插值**：用IMU高频特性对其他传感器数据进行时间插值
4. **在线时间偏移估计**：将时间偏移 $t_d$ 作为优化变量加入状态估计（如VINS-Mono中的td估计）
5. **互相关法**：利用IMU角速度与相机视觉角速度的互相关来估计时间差

**实际做法：** 硬件同步 > 在线估计td > 时间插值。常将时间偏移建模为优化变量在初始化阶段标定。

**参考：**
- [Kalibr - VI传感器标定工具](https://github.com/ethz-asl/kalibr)
- [VINS-Mono论文 - online temporal calibration](https://arxiv.org/abs/1708.03852)
- [LI-Init - LiDAR-IMU初始化](https://github.com/hku-mars/LiDAR_IMU_Init)

---

### 5. 如何理解IMU的噪声？IMU的测量噪声和零偏各自属于什么数学模型？

**解答：**

**IMU测量噪声（Measurement Noise / White Noise）：**
- **数学模型**：高斯白噪声（连续时间）$\sim \mathcal{N}(0, \sigma^2)$
- **离散化后**：$\sigma_d = \sigma \cdot \frac{1}{\sqrt{\Delta t}}$（白噪声的离散方差与采样频率有关）
- **来源**：传感器电子热噪声、ADC量化噪声
- **参数名**：加速度计"noise density"（单位 $\frac{m/s^2}{\sqrt{Hz}}$）、陀螺仪"noise density"（单位 $\frac{rad/s}{\sqrt{Hz}}$ 或 $\frac{deg/s}{\sqrt{Hz}}$）
- **数学表达**：连续白噪声过程 $\eta(t) \sim \mathcal{N}(0, \sigma^2 \cdot I)$

**IMU零偏（Bias）：**
- **数学模型**：随机游走过程（Random Walk / Wiener Process / Brownian Motion）
- **微分方程**：$\dot{b}(t) = \eta_b(t)$，其中 $\eta_b(t)$ 是白噪声
- **离散化后**：$b_k = b_{k-1} + \eta_{b,k}$，$\eta_{b,k} \sim \mathcal{N}(0, \sigma_{b}^2 \cdot \Delta t)$
- **来源**：温度变化、机械应力、电子元件老化
- **参数名**：加速度计"bias instability"、"random walk"（单位 $\frac{m/s^2}{\sqrt{h}}$ 或换算后）、陀螺仪"bias instability"（单位 $^\circ/h$ 或 $^\circ/\sqrt{h}$）

**IMU完整测量模型：**

加速度计：$a_m = a_{true} + b_a + \eta_a$

陀螺仪：$\omega_m = \omega_{true} + b_g + \eta_g$

其中 $b_a, b_g$ 为随机游走bias，$\eta_a, \eta_g$ 为白噪声。

**参考：**
- [IMU Noise Model - Kalibr Wiki](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model)
- [A micro PIG for state estimation - IMU预积分](https://arxiv.org/abs/1512.02363)

---

### 6. 怎么实现IMU的初始化校准和重力校准？IMU技术规格书中有哪些参数可以指导配置状态估计算法的权重？

**解答：**

**IMU初始化校准：**

1. **静态采集**：将IMU静置（不同朝向），采集数分钟数据
2. **加速度计校准**：利用静止时加速度计只测重力（$\|a\| = g$），用六面法或椭球拟合估计scale、misalignment和bias
3. **陀螺仪校准**：利用静止时陀螺仪应读数为0，通过Allan方差分析确定噪声参数
4. **工具**：imu_utils（港科大）、Kalibr中的IMU标定

**重力校准：**
1. 初始静止阶段，用加速度计均值估计重力方向
2. 将估计的重力方向与预设重力 $[0, 0, -g]^T$ 对齐，计算初始姿态
3. VIO初始化中用IMU预积分和视觉SfM联合估计重力（如VINS-Mono的初始化流程）

**IMU规格书中的关键参数：**

| 参数 | 英文名 | 用途 |
|------|--------|------|
| 噪声密度 | Noise Density / Angular Random Walk (ARW) / Velocity Random Walk (VRW) | 配置 Q 矩阵的离散噪声项 |
| 零偏不稳定性 | Bias Instability | 配置 Q 矩阵的bias随机游走项 |
| 零偏重复性 | Bias Repeatability | 初始化bias的先验不确定性 |
| 带宽 | Bandwidth | 判断滤波/预积分的截止频率 |
| 量程 | Full Scale Range | 防止饱和 |
| 非线性度 | Non-linearity | 评估模型准确性 |

**权重配置指导：**
- Noise Density → 过程噪声 Q 的白噪声部分：$Q_{noise} \propto \sigma^2 \cdot \Delta t$
- Bias Instability → 过程噪声 Q 的随机游走部分：$Q_{bias} \propto \sigma_b^2 \cdot \Delta t$
- 噪声越大 → Q越大 → KF更信任观测

**参考：**
- [imu_utils - IMU标定工具](https://github.com/gaowenliang/imu_utils)
- [Allan Variance 教程](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model)

---

### 7. 简述几种常见LiDAR传感器的数据获取原理？机械式和固态式LiDAR的优缺点？

**解答：**

**机械旋转式LiDAR（Mechanical LiDAR）：**
- **原理**：通过电机旋转带动激光收发模组，360°扫描环境。每个激光通道有不同的俯仰角，水平方向通过旋转获得。
- 使用ToF（Time of Flight）测量距离：$d = \frac{c \cdot \Delta t}{2}$
- **代表**：Velodyne HDL-64E, VLP-16, Hesai Pandar, RoboSense RS-LiDAR

**固态LiDAR（Solid-State LiDAR）：**
- **MEMS振镜式**：用MEMS微镜偏转光束扫描，代表：Innoviz, RoboSense M1
- **Flash LiDAR**：一次性发射面阵激光（类似相机闪光灯），一次性获取整幅深度图，代表：Ouster ES2
- **OPA（光学相控阵）**：通过调节波导间相位差控制出射光束方向，代表：Quanergy S3

**半球形/非重复扫描LiDAR：**
- **原理**：通过棱镜旋转产生非重复的扫描模式（玫瑰花图），积分时间越长分辨率越高
- **代表**：Livox Mid-40, Livox Horizon, Livox Avia

**优缺点对比：**

| 特性 | 机械式 | MEMS | Flash | 非重复扫描 |
|------|--------|------|-------|-----------|
| 视场角 | 360° | 120° | 窄 | 70°-81° |
| 分辨率 | 固定 | 可调 | 阵列固定 | 积分可增加 |
| 可靠性 | 低（运动部件） | 中 | 高 | 中 |
| 成本 | 高 | 中低 | 低 | 中 |
| 测距 | 远(200m+) | 中(150m) | 短(50m) | 中(260m) |
| 车规级 | 难 | 容易 | 容易 | 量产中 |

**参考：**
- [LiDAR 技术综述](https://ieeexplore.ieee.org/document/8978955)
- [Livox LiDAR 技术文档](https://www.livoxtech.com/)

---

### 8. GNSS/RTK的数据有什么特点？

**解答：**

**GNSS数据特点：**
- **全球性**：全球覆盖，全天候可用
- **绝对定位能力**：提供WGS84坐标系下绝对位置（经纬高）
- **无累积误差**（核心优势）：不像IMU/VO/LIO会漂移，时间越长累积误差越大
- **频率低**：通常1-20Hz（远低于IMU的100-1000Hz）
- **易受环境影响**：城市峡谷、隧道、高架桥下多路径效应严重，信号被遮挡
- **精度有限**：单点定位精度米级（2-5m）

**RTK（Real-Time Kinematic）特点：**
- **精度显著提高**：通过差分修正，精度可达**厘米级（1-2cm）**
- **需要基站**：需要已知位置的基站或CORS网络（NTRIP协议）
- **需要通信**：通过4G/电台接收差分改正数
- **收敛时间**：需要一定时间获得固定解（Fix Solution）
- **可靠性**：RTK固定解 vs 浮点解，浮点解精度大大下降
- **局限性**：遮挡环境下RTK也很容易变成浮点解

**在SLAM中的角色：**
- 作为全局约束消除累积漂移（绝对参考）
- 通常作为因子图优化中的**先验因子（Prior Factor）**或**观测因子**
- 只有RTK Fix解时才可靠，Float解需要降权或忽略

**参考：**
- [RTKLIB - GNSS开源定位库](http://www.rtklib.com/)
- [GNSS-SIM, GNSS-IMU-仿真](https://github.com/weisongwen/GINS)

---

### 9. 常见的GNSS误差有哪些？会如何影响GNSS数据？

**解答：**

| 误差来源 | 影响程度 | 特性 | 误差量级 |
|---------|---------|------|---------|
| **电离层延迟** | 大 | 与频率有关，双频可消除一阶项 | 5-50m |
| **对流层延迟** | 中 | 与温湿压有关，与频率无关 | 0.5-2m |
| **卫星钟差** | 大 | 可通过差分消除 | 米级 |
| **卫星轨道误差** | 中 | 广播星历vs精密星历 | 1-5m（广播） |
| **多路径效应** | 大（城市） | 信号反射导致，最难消除 | 米到十米级 |
| **接收机噪声** | 小 | 热噪声，随机 | 0.1-1m |
| **卫星几何分布** | 不可忽视 | PDOP/GDOP值，几何构型差放大了测距误差 | DOP系数 |

**影响机制：**
- **电离层/对流层**：改变信号传播速度 → 伪距误差
- **多路径**：产生错误的相关峰 → 码相位和载波相位都受影响 → 难以用差分消除
- **卫星几何**：所有卫星聚在一起 → 定位精度几何放大
- **遮挡**：可见卫星数不足 → 无法定位或精度急剧下降

**参考：**
- [GPS Error Analysis](https://gssc.esa.int/navipedia/index.php/GPS_Error_Sources)
- [RTKLIB Manual](http://www.rtklib.com/prog/manual_2.4.2.pdf)

---

### 10. 如何规避GNSS误差对多传感器融合系统的影响？

**解答：**

1. **质量评估与拒绝机制**：
   - 仅使用 RTK Fix 解，拒绝 Float 解和 Single 解
   - 检查协方差/标准差（GNSS接收机通常输出）是否超过阈值
   - 检查可见卫星数、DOP值

2. **鲁棒优化方法**：
   - **Huber核函数**：对大残差的GNSS因子进行压制
   - **Cauchy核函数**：更强的抗差效果
   - **动态信息矩阵缩放**：根据GNSS质量动态调整因子权重

3. **多源融合策略**：
   - GNSS + IMU（松/紧耦合）：有GNSS时校正IMU，无GNSS时IMU递推（INS）
   - GNSS + LiDAR/Visual SLAM：当GNSS失效时，用LiDAR/Visual局部约束维持精度
   - GNSS因子添加条件：仅在RTK Fix + 高俯仰角卫星数足够时加入

4. **工程技巧**：
   - 设置水平和垂直方向不同的阈值
   - 多路径检测（C/N0 低于阈值排除）
   - 使用时间窗口平滑GNSS（避免单个异常值）
   - 建立GNSS观测残差的卡方检验门控

**参考：**
- [VINS-Fusion - GNSS-Visual-Inertial融合](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion)
- [GINS - GNSS/INS紧耦合](https://github.com/weisongwen/GINS)

---

### 11. 你如何理解图优化方法和滤波方法？两者的优缺点？

**解答：**

**滤波方法（Filtering-based）：**
- **核心思想**：递归维护当前后验，并将历史信息压缩在状态与协方差中。具体系统也可能维护有限历史状态，例如MSCKF的相机位姿克隆。
- **代表**：MSCKF, ROVIO, OpenVINS
- **数学**：$p(x_k | z_{1:k})$ — 只估计当前状态

**图优化方法（Graph Optimization / Smoothing-based）：**
- **核心思想**：维护全部或滑动窗口内所有历史状态，构建因子图/位姿图进行全局优化
- **代表**：ORB-SLAM, VINS-Mono, LIO-SAM, GTSAM/iSAM2
- **数学**：$p(X | Z)$ — 估计全轨迹

**优缺点对比：**

| 维度 | 滤波方法 | 图优化方法 |
|------|---------|-----------|
| **计算复杂度** | 取决于状态维度和观测维度；协方差更新常含稠密矩阵运算 | 取决于变量连接稀疏性、消元顺序和求解器；不能简单概括为固定$O(n^3)$ |
| **精度** | 一次线性化（一阶） | 多次迭代线性化（更高精度） |
| **历史修正** | 已边缘化历史通常不能直接重优化，但可在外部位姿图中做全局修正 | 可保留或重新线性化窗口/全局状态，回环后可整体调整 |
| **内存** | 恒定（只存当前状态） | 随轨迹增长（需边缘化/滑动窗口） |
| **实现复杂度** | 相对简单 | 较复杂（建模因子图） |
| **适用场景** | 资源受限嵌入式系统 | 追求精度的建图/定位系统 |
| **线性化误差** | 累积（一阶近似） | 迭代优化减少 |

**实际趋势：** 现代SLAM系统倾向于图优化（精度优势明显），但滤波在计算资源受限场景仍有价值。

**参考：**
- [Factor Graphs for Robot Perception (Dellaert & Kaess, 2017)](https://ieeexplore.ieee.org/document/8070881)
- [iSAM2: Incremental Smoothing and Mapping](https://ieeexplore.ieee.org/document/6096436)

---

### 12. 图优化方法和滤波方法本质上可以看作同一个数学模型吗？怎么证明？

**解答：**

**可以。** 从概率角度，两者都是**最大后验估计（MAP）**的具体实现，在高斯噪声假设下等价于**非线性最小二乘**。

**证明思路：**

**滤波方法（以EKF为例）：**
$$p(x_k | z_{1:k}) \propto p(z_k | x_k) \cdot p(x_k | z_{1:k-1})$$

取负对数并假设高斯噪声：
$$-\log p(x_k | z_{1:k}) \propto \|z_k - h(x_k)\|^2_{R_k} + \|x_k - f(x_{k-1})\|^2_{P_{k|k-1}}$$

**图优化方法（全平滑）：**
$$p(X | Z) \propto \prod_k p(z_k | x_k) \prod_k p(x_k | x_{k-1})$$

取负对数：
$$-\log p(X | Z) \propto \sum_k \|z_k - h(x_k)\|^2_{R_k} + \sum_k \|x_k - f(x_{k-1})\|^2_{Q_k}$$

**关键联系：**
1. 滤波在**每个时刻**求解一个MAP问题（边缘化掉历史）
2. 图优化在**所有时刻**求解一个更大的MAP问题
3. **全平滑的最后一个变量的边缘后验** 等于 **滤波的后验**（在无回环时）
4. EKF本质上做了**一次高斯-牛顿迭代**，而图优化可以做**多次迭代**

**严格证明：** 可以用因子图+消元法（变量消元）来证明：全平滑因子图上执行变量消元后，消元结果的后验边缘等价于滤波的递归更新。具体来说，EKF/Schur补消元等价于在因子图上按时间顺序逐一边缘化变量。

**参考：**
- [Factor Graphs and GTSAM (Dellaert)](https://gtsam.org/tutorials/)
- [Simultaneous Localization and Mapping: A Survey (Cadena et al., 2016)](https://ieeexplore.ieee.org/document/7485862)

---

### 13. 是否熟悉ceres、g2o或gtsam等优化库？如何自定义构建残差方程？

**解答：**

（这个问题需要根据面试者实际使用经验回答，以下为通用知识点。）

**Ceres Solver：**
```cpp
// 自定义残差：继承SizedCostFunction或AutoDiffCostFunction
struct MyResidual {
    template <typename T>
    bool operator()(const T* const param, T* residual) const {
        residual[0] = ...; // 残差计算
        return true;
    }
};
// 构建问题
ceres::Problem problem;
ceres::CostFunction* cost = new ceres::AutoDiffCostFunction<MyResidual, 1, 6>(new MyResidual());
problem.AddResidualBlock(cost, loss_function, param);
```

**g2o：**
```cpp
// 自定义边：继承BaseEdge或BaseUnaryEdge
class MyEdge : public g2o::BaseUnaryEdge<1, double, VertexSE3> {
    void computeError() override {
        const VertexSE3* v = static_cast<const VertexSE3*>(_vertices[0]);
        _error[0] = ... ; // 误差计算
    }
};
optimizer.addEdge(edge);
```

**GTSAM：**
```cpp
// 自定义因子：继承NoiseModelFactor
class MyFactor : public gtsam::NoiseModelFactor1<Pose3> {
    Vector evaluateError(const Pose3& pose, boost::optional<Matrix&> H) const override {
        // 返回误差，可选Jacobian
        return ...;
    }
};
graph.add(factor);
```

**参考：**
- [Ceres Solver Tutorial](http://ceres-solver.org/tutorial.html)
- [g2o GitHub](https://github.com/RainerKuemmerle/g2o)
- [GTSAM Tutorials](https://gtsam.org/tutorials/)

---

### 14. 你怎么理解紧耦合和松耦合的优化方法？各自的优缺点？

**解答：**

**松耦合（Loose Coupling）：**
- 各传感器独立估计位姿/状态，然后再融合结果
- 例如：先用LiDAR做ICP得到位姿，用视觉做VO得到位姿，再对两个位姿做加权平均或EKF
- 各模块独立运行，互不干扰

**紧耦合（Tight Coupling）：**
- 在同一个优化框架中联合使用所有传感器的原始观测
- 例如：将IMU预积分因子、视觉重投影因子、LiDAR点面距离因子放在同一个因子图中联合优化
- 各传感器原始数据互相约束

**优缺点对比：**

| 维度 | 松耦合 | 紧耦合 |
|------|--------|--------|
| **精度** | 较低（丢失传感器间互补信息） | 高（充分利用传感器间耦合关系） |
| **鲁棒性** | 较好（一个传感器失效不影响其他） | 需处理某传感器失效的情况 |
| **实现复杂度** | 低 | 高 |
| **系统耦合度** | 低（模块化） | 高 |
| **调参难度** | 低 | 高 |
| **代表系统** | 早期SLAM系统 | VINS-Mono, LIO-SAM, FAST-LIO2 |

**当前趋势：** 紧耦合是主流方案，因为可以利用IMU的高频运动信息直接约束视觉/LiDAR的位姿估计，精度和鲁棒性都优于松耦合。

**参考：**
- [VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator](https://arxiv.org/abs/1708.03852)
- [LIO-SAM: Tightly-coupled Lidar Inertial Odometry via Smoothing and Mapping](https://arxiv.org/abs/2007.00258)

---


## 四、视觉SLAM与视觉惯性里程计VIO

### 1. 相机针孔模型是什么？从世界点到像素点经历哪些变换？【高频】

世界点 $P_w$ 先通过相机位姿变换到相机坐标系：

$$P_c=R_{cw}P_w+t_{cw}=[X,Y,Z]^T$$

归一化平面：

$$x=X/Z,\qquad y=Y/Z$$

再通过内参：

$$u=f_xx+c_x,\qquad v=f_yy+c_y$$

矩阵形式：

$$s\begin{bmatrix}u\\v\\1\end{bmatrix}=K[R|t]\begin{bmatrix}P_w\\1\end{bmatrix}$$

**面试追问：为什么 $Z\le0$ 的点不能正常投影？** 因为它位于相机后方或投影深度无效。

---

### 2. 常见相机畸变模型有哪些？

- Brown-Conrady：径向 $k_1,k_2,k_3$ 和切向 $p_1,p_2$；
- Fisheye/Kannala-Brandt：适合大视场鱼眼；
- Unified/Double Sphere：适合全向相机；
- Mei模型：常用于catadioptric系统。

工程中必须确认：标定模型与投影代码一致、图像是否已去畸变、参数顺序和单位是否正确。模型用错会表现为图像边缘重投影误差大、轨迹弯曲或初始化失败。

---

### 3. 本质矩阵E、基础矩阵F和单应矩阵H有什么区别？【高频】

- $E$：归一化相机坐标之间的极线约束，已知内参；$E=[t]_\times R$；
- $F$：像素坐标之间的极线约束，包含内参；$F=K_2^{-T}EK_1^{-1}$；
- $H$：平面场景或纯旋转条件下的单应变换。

极线约束：

$$x_2^TEx_1=0,\qquad p_2^TFp_1=0$$

**追问：什么时候H比E更合适？** 主要平面、远景近似平面或相机纯旋转时。

---

### 4. 为什么五点法、八点法需要RANSAC？

特征匹配中存在错误对应。最小解法只需要少量点，但对离群点非常敏感。RANSAC随机采样模型、计算内点集合，再用全部内点精化。

面试应说明四个参数：最小样本数、内点阈值、最大迭代数、置信度。阈值应与归一化坐标或像素误差单位匹配。

---

### 5. 三角化的原理和退化条件是什么？【高频】

已知两个相机位姿和对应像素射线，寻找最接近两条射线的三维点。可用DLT构造齐次线性方程，也可最小化重投影误差。

退化情况：

- 基线过小或视差过小；
- 纯旋转；
- 匹配错误；
- 点在相机后方；
- 远点导致深度不确定性很大。

工程判断：正深度检查、视差阈值、三角化夹角、重投影误差、深度范围。

---

### 6. PnP解决什么问题？常见方法有哪些？

给定三维点及其二维像素观测，估计相机位姿。

- P3P：最少3组点，产生多个候选解；
- EPnP：效率高，适合较多点；
- DLS/UPnP：不同参数化；
- RANSAC-PnP：抗错误匹配；
- 非线性PnP：以重投影误差进一步优化。

ORB-SLAM等系统常用PnP做重定位和初始位姿估计，再做motion-only BA精化。

---

### 7. 特征法、直接法和半直接法的区别？【高频】

**特征法：** 检测角点、描述子匹配，后端最小化重投影误差。优点是对光照变化相对鲁棒、适合回环；缺点是低纹理环境困难。

**直接法：** 直接最小化像素光度误差，利用更多像素信息。优点是弱纹理时可能更好；缺点是依赖光度一致性、曝光和初值。

**半直接法：** 前端直接法跟踪，后端特征几何优化，例如SVO类方法。

---

### 8. KLT光流的基本假设和失败场景是什么？

三大假设：亮度恒定、小运动、局部邻域运动一致。通过图像金字塔处理较大位移，通过前后向检查和RANSAC剔除异常轨迹。

失败场景：运动模糊、曝光突变、大遮挡、重复纹理、低纹理、动态物体、帧率过低。

---

### 9. ORB特征为什么适合SLAM？

ORB由带方向的FAST关键点和旋转BRIEF描述子组成，二进制描述子可用Hamming距离快速匹配。它适合实时CPU、回环和重定位。

缺点：尺度和视角变化很大时不如更重的局部特征；运动模糊、弱纹理和重复纹理下仍会失败。

---

### 10. Bundle Adjustment优化什么？Local BA、Global BA和Motion-only BA的区别？【高频】

BA联合优化相机位姿和三维路标，使总重投影误差最小。

- Motion-only BA：固定地图点，只优化当前相机位姿；
- Local BA：优化局部关键帧和局部地图点，固定边界关键帧；
- Global BA：优化全部关键帧和地图点，精度高但计算重。

回环后通常先位姿图快速校正，再异步运行更重的全局BA。

---

### 11. 单目SLAM为什么没有绝对尺度？如何恢复？【高频】

单目投影对同时缩放三维结构和平移不敏感，因此只能恢复相似变换意义下的地图。尺度可通过：

- IMU和重力；
- 已知物体尺寸；
- 相机高度或地面约束；
- 轮速/GNSS；
- 双目基线或RGB-D深度。

---

### 12. VIO的状态通常包含什么？哪些方向不可观？【高频】

典型状态：

$$x=[R_{wi},p_{wi},v_{wi},b_g,b_a]$$

还可包括相机—IMU外参、时间偏移、重力、尺度和特征深度。

标准VIO在充分激励下可恢复尺度和重力方向，但全局位置3维、绕重力轴的全局yaw 1维不可观。

---

### 13. IMU预积分解决了什么问题？【高频】

如果每次优化改变起始状态都重新积分全部高频IMU，代价很高。预积分把两关键帧之间大量IMU测量压缩成相对旋转、速度和位置增量及其协方差，并在某个bias线性化点附近用Jacobian做一阶bias修正。

优化变量变化较小时不用重积分；bias变化过大时需要重新预积分或repropagation。

---

### 14. VIO初始化要估计哪些量？为什么难？

通常需要估计：

- 视觉结构和相机相对位姿；
- 尺度；
- 重力方向；
- 初始速度；
- 陀螺仪和加速度计bias；
- 可选外参和时间偏移。

难点在于多变量耦合和运动激励。纯旋转、匀速、短时间或低视差会使尺度、重力和bias难以区分。

---

### 15. 什么运动有利于VIO初始化？

需要同时具有：

- 足够平移视差；
- 多轴旋转，激励陀螺仪外参和bias；
- 非恒定加速度，帮助区分重力与线性加速度；
- 避免长时间静止、纯旋转和单轴匀速。

工程上可设计“8字运动”或多方向平移加旋转。

---

### 16. 滑动窗口VIO为什么要边缘化？边缘化哪一帧？

为保证实时性，窗口大小固定。常见策略：

- 新帧是关键帧：边缘化最老关键帧及其弱连接路标；
- 新帧不是关键帧：边缘化次新帧，保留关键帧结构；
- 根据共视关系、视差、特征数量和时间跨度选择边缘化对象。

边缘化策略影响数值稳定性、信息保留和计算量。

---

### 17. MSCKF的核心思想是什么？与滑窗BA有什么区别？【高频】

MSCKF将多个历史相机位姿克隆到ESKF状态中。一条特征跨多帧观测，先临时三角化，再通过零空间投影消除特征误差，用剩余残差更新所有相关clone。

与滑窗BA相比：

- MSCKF维护状态和联合协方差，通常一次线性化更新；
- BA显式维护窗口变量，可多次重新线性化；
- MSCKF低延迟、计算可预测；
- BA精度上限和添加新因子的灵活性通常更高。

---

### 18. Basalt的核心代码流程怎么概括？

Basalt是固定时滞的优化式VIO，不是全历史full batch，也不是EKF。主要流程：

1. patch optical flow跟踪特征；
2. 相邻图像间IMU预积分；
3. `predictState()`提供新状态初值；
4. 加入重投影、IMU、bias和边缘化先验；
5. 线性化并通过QR/Schur结构消除路标；
6. 求解相机和惯性状态，再回代路标；
7. 优化后边缘化旧状态，保留square-root prior。

**面试关键词：** fixed-lag smoother、square-root marginalization、QR landmark elimination、pose-only keyframe。

---

### 19. VINS-Mono的整体流程和贡献是什么？【高频】

- KLT光流跟踪和关键帧选择；
- 单目视觉SfM与视觉惯性对齐初始化；
- IMU预积分；
- 滑动窗口中优化位姿、速度、bias、逆深度及可选外参/时间偏移；
- 边缘化保持固定计算量；
- DBoW回环和4DoF位姿图修正位置与yaw漂移。

它更像“高精度局部VIO + 全局位姿图校正”，不是ORB-SLAM3那种长期地图驱动系统。

---

### 20. ORB-SLAM3的核心架构是什么？

主要线程：Tracking、Local Mapping、Loop Closing，以及Atlas多地图管理。视觉惯性模式中加入IMU预积分、惯性初始化和视觉惯性BA。

优势：完整SLAM、重定位、回环、多地图融合、单目/双目/RGB-D和鱼眼支持。

不足：系统复杂、状态机多、代码改造成本较高；动态和弱纹理场景仍可能失败。

---

### 21. OpenVINS的核心特点是什么？

OpenVINS是以ESKF/MSCKF为核心的滤波式VIO平台，维护IMU状态、相机clone、联合协方差和可选SLAM特征。强调：

- FEJ和一致性；
- 零空间投影；
- 在线相机内外参、时间偏移及IMU内参标定；
- KLT/描述子前端；
- 低延迟和协方差输出。

核心并不是完整的全局地图和回环系统。

---

### 22. ORB-SLAM3、VINS-Mono、OpenVINS、Basalt如何选？【高频】

| 系统 | 后端 | 主要目标 | 典型选择理由 |
|---|---|---|---|
| ORB-SLAM3 | 局部/全局优化 | 完整SLAM | 回环、重定位、地图复用 |
| VINS-Mono | 滑窗非线性优化 | 单目VIO | 易加因子、研究预积分与边缘化 |
| OpenVINS | ESKF/MSCKF | 实时滤波VIO | 协方差、在线标定、一致性 |
| Basalt | square-root滑窗优化 | 高效VIO/VO | QR消点、数值稳定、现代C++实现 |

不能只按“哪个精度最高”选择，要结合传感器、算力、回环需求、地图类型和可维护性。

---

### 23. 回环检测、重定位和地图融合有什么区别？

- 回环检测：判断当前位置是否访问过历史区域；
- 重定位：当前跟踪丢失后，根据已有地图重新求当前位姿；
- 地图融合：将两个独立子地图对齐并合并共视结构。

三者都需要地点识别，但后续几何验证、图结构修改和优化目标不同。

---

### 24. 回环如何避免误检？

常用多级验证：

1. BoW/全局描述子候选检索；
2. 时间和共视一致性检查；
3. 局部特征匹配；
4. PnP/Sim3/RANSAC几何验证；
5. 足够内点和空间分布；
6. 加入鲁棒位姿图并进行switchable constraint或一致性检查。

错误回环属于灾难性异常，不能只依赖Huber核。

---

### 25. Rolling Shutter对VIO有什么影响？如何处理？

滚动快门一帧内不同行在不同时间曝光，快速运动时不能用单一相机位姿解释整张图像，会造成弯曲、重投影误差和错误尺度。

处理方式：提高快门速度、使用全局快门、将行时间纳入投影模型、用IMU连续轨迹对每个特征按曝光时间补偿。

---

### 26. 相机—IMU时间偏移如何在线估计？

将时间偏移 $t_d$ 作为状态。视觉观测对应的相机姿态近似在 $t+t_d$ 处，通过速度或IMU传播建立对 $t_d$ 的Jacobian。

可观性依赖运动：静止或匀速时很难估计，丰富角运动最有效。在线估计只能补偿近似常量偏移，无法解决随机抖动和系统性丢帧。

---

### 27. 动态场景为什么会破坏视觉SLAM？如何处理？

传统视觉SLAM假设大部分观测来自静态世界。动态物体会产生错误光流、错误三角化和虚假回环。

方法：

- 几何RANSAC与重投影残差门控；
- 多模型运动分割；
- 语义掩码；
- 时序一致性和场景流；
- 对疑似动态特征降权而不是全部删除；
- 地图层维护静态概率和可见性。

---

### 28. 如何评价VO/VIO/SLAM轨迹？ATE和RPE有什么区别？

- ATE：轨迹对齐后比较绝对位姿，反映全局一致性和累计漂移；
- RPE：比较固定时间/距离间隔的相对运动，反映局部里程计误差；
- 单目常用Sim(3)对齐，带尺度系统通常用SE(3)对齐；
- 还应报告跟踪成功率、运行时间、内存、回环前后结果和多次运行方差。

不能只报单个序列的最好结果，也不能把不同对齐方式下的ATE直接横向比较。



## 五、点云配准

### 1. 点云配准中的三大假设是什么？

**解答：**

1. **最近点对应假设（Closest Point Correspondence）**：源点云中每个点在目标点云中的对应点是其欧式距离最近的点。这是ICP等算法的基础假设。

2. **刚性变换假设（Rigid Transformation）**：两点云之间的变换是刚性的（只涉及旋转和平移），不考虑形变。即 $p' = R p + t$。

3. **充分重叠假设（Sufficient Overlap）**：两个点云之间有足够的重叠区域（overlap ratio），以保证能找到足够的有效对应点。

**为何需要这些假设：**
- 没有假设1 → 无法建立对应关系
- 没有假设2 → 需要引入非刚性配准，问题复杂得多
- 没有假设3 → 对应关系不可靠，配准会发散

**实际应对策略：**
- 假设1不满足时 → 使用特征匹配（FPFH等）作为初始对应
- 假设2不满足时 → 使用非刚性ICP（Non-rigid ICP/CPD）
- 假设3不满足时 → 增大传感器视场角、降低运动速度、使用回环检测

**参考：**
- [ICP原始论文 (Besl & McKay, 1992)](https://ieeexplore.ieee.org/document/121791)
- [Point Cloud Registration: A Survey](https://arxiv.org/abs/1910.12247)

---

### 2. 点云配准算法的基本流程是什么？

**解答：**

**标准流程（以ICP为例）：**

**1. 预处理/滤波（Preprocessing）：**
- 降采样（Voxel Grid Filter）减少点数，均匀化密度
- 去噪（Statistical Outlier Removal / Radius Outlier Removal）
- 法向量估计

**2. 对应点搜索（Correspondence Search）：**
- 最近邻搜索（KNN / KD-Tree）
- 法向量一致性过滤（法向量方向差异过大的去除）
- 双向对应过滤（互相互为最近点）
- 最大距离阈值过滤

**3. 异常对应剔除（Outlier Rejection）：**
- 距离阈值过滤
- 中值距离的倍数阈值
- RANSAC

**4. 变换估计（Transformation Estimation）：**
- SVD分解法（Arun方法）
- 最小二乘/加权最小二乘

**5. 收敛判断：**
- $\| \Delta T \| < \epsilon$（变换增量小于阈值）
- 最大迭代次数
- RMSE变化小于阈值

**6. 变换更新：** $T \leftarrow \Delta T \cdot T$

**参考：**
- [PCL ICP教程](https://pcl.readthedocs.io/projects/tutorials/en/latest/iterative_closest_point.html)
- [ICP变种综述 (Pomerleau et al., 2015)](https://ieeexplore.ieee.org/document/6942309)

---

### 3. ICP、GICP和NDT的差异和优缺点？

**解答：**

**ICP (Iterative Closest Point)：**
- **原理**：基于点到点距离最小化
- **误差函数**：$\min_{R,t} \sum_i \| (R p_i + t) - q_i \|^2$
- **优点**：简单，收敛快（有良好初值时）
- **缺点**：对初值敏感，容易陷入局部最优；对离群点敏感

**Point-to-Plane ICP：**
- **原理**：基于点到平面距离最小化
- **误差函数**：$\min_{R,t} \sum_i | (R p_i + t - q_i)^T n_i |^2$
- **优点**：收敛速度更快（允许沿平面的滑动），精度更高
- **缺点**：需要法向量；平地面对旋转约束弱（需要足够结构特征）

**GICP (Generalized ICP)：**
- **原理**：统一了Point-to-Point和Point-to-Plane的框架，将两点云视为概率分布
- **误差函数**：$\min_{R,t} \sum_i d_i^T (C_i^B + R C_i^A R^T)^{-1} d_i$，其中 $d_i = q_i - (R p_i + t)$
- **优点**：自适应地选择Point-to-Point或Point-to-Plane（根据局部协方差）；鲁棒性更好
- **缺点**：计算量较大（需要计算局部协方差矩阵）

**NDT (Normal Distributions Transform)：**
- **原理**：将点云划分体素网格，每个体素用3D高斯分布表示，配准是最大化源点在目标NDT中的概率
- **优点**：不需要明确的对应搜索（直接对概率求和）；平滑性好；适合大规模点云
- **缺点**：对体素大小敏感；不适合稀疏点云

**综合对比：**

| 方法 | 精度 | 速度 | 鲁棒性 | 对初值敏感度 |
|------|------|------|--------|-------------|
| Point-to-Point ICP | 中 | 快 | 低 | 高 |
| Point-to-Plane ICP | 高 | 快 | 中 | 中 |
| GICP | 高 | 中 | 高 | 中 |
| NDT | 中 | 中 | 中高 | 中；仍需要合理初值和合适体素尺度 |

**实际应用经验：**
- 城市环境有丰富平面结构 → Point-to-Plane ICP / GICP
- 初值差/大范围 → NDT
- 追求效率 → Point-to-Plane ICP + 多分辨率

**参考：**
- [Generalized-ICP (Segal et al., 2009)](http://www.roboticsproceedings.org/rss05/p21.pdf)
- [NDT - The Normal Distributions Transform (Biber & Strasser, 2003)](https://ieeexplore.ieee.org/document/1249285)
- [PCL Registration模块](https://pcl.readthedocs.io/projects/tutorials/en/latest/#registration)

---

### 4. 影响点云配准精度的关键因素？

**解答：**

1. **点云质量**：密度、噪声水平、分辨率
2. **初始位姿（Initial Guess）**：差的初值导致ICP收敛到局部最优
3. **几何结构丰富度**：平面场景退化解（沿平面方向不确定），有丰富三维结构的场景精度高
4. **离群点/动态物体**：运动物体、遮挡产生的虚假点
5. **重叠率（Overlap）**：重叠不足 → 对应关系错误
6. **对应点选择策略**：最近点 vs 法向量过滤 vs 特征匹配
7. **传感器自身误差**：测距精度、角度分辨率（机械LiDAR的编码器误差）
8. **LiDAR的运动畸变**：快速运动时扫描帧内的点不在同一时刻采集

**参考：**
- [Comparing ICP Variants on Real-World Data Sets (Pomerleau et al., 2013)](https://link.springer.com/article/10.1007/s10514-013-9327-2)

---

### 5. 影响点云配准鲁棒性的关键因素？

**解答：**

1. **动态障碍物**：行人、车辆等运动物体产生"幽灵点"
2. **视点变化**：同一场景从不同角度观测，局部几何有明显差异
3. **环境退化**：长廊（长走廊）、隧道、广阔平地（几何约束不足）
4. **传感器退化**：反光/吸光表面、雾/雨/雪天气
5. **离群点比例**：遮挡、边缘效应、反射导致的大量离群点
6. **LiDAR运动畸变**：快速旋转或运动时帧内畸变
7. **异常对应剔除**：没有有效的Outlier Rejection策略
8. **二次结构/对称场景**：如高度对称的室内走廊，多个可能解

**改进方法：**
- 对退化环境：检测退化维度（用特征值分析），在退化方向降低权重或引入额外约束
- 对动态物体：基于语义分割/运动检测剔除动态点
- 对离群点：使用鲁棒核函数（Huber, Cauchy）或RANSAC

**参考：**
- [LOAM: Lidar Odometry and Mapping in Real-time](https://www.ri.cmu.edu/pub_files/2014/7/Ji_LidarMapping_RSS2014_v8.pdf)
- [Degeneracy-aware LiDAR SLAM (Zhang et al.)](https://arxiv.org/abs/2011.02341)

---

### 6. 你有使用过哪种点云配准算法？你的使用感受是什么？

**解答（参考回答框架）：**

- 实际经验中常用的是**Point-to-Plane ICP**（PCL实现）和**GICP**（Fast-GICP实现）
- Point-to-Plane ICP在室内结构化环境表现良好，收敛快，但室外非结构化场景精度下降
- GICP在大部分场景表现最好，是平衡精度和效率的选择
- NDT在全局定位（重定位）场景表现出色，对初值不敏感
- **Fast-GICP**（Koide et al., 2021）通过多线程和体素化加速GICP，实现了实时性能
- 实际工程中预处理（体素降采样、离群剔除）对配准质量的影响往往比算法选择更大

**参考：**
- [Fast-GICP (Koide et al., 2021)](https://github.com/SMRT-AIST/fast_gicp)
- [Small, Voxelized, and Optimized GICP](https://arxiv.org/abs/2109.09755)

---

### 7. 如何剔除动态障碍物对点云配准的负面影响？你熟悉哪些离群点滤除算法？

**解答：**

**动态障碍物剔除方法：**

1. **基于一致性的方法**：
   - 配准后检查每个对应点的残差，残差超过阈值的标记为离群点（动态对象产生的匹配残差大）

2. **基于语义分割**：
   - 用RangeNet++、Cylinder3D等对点云进行语义分割
   - 标记为"car", "pedestrian"的类别剔除

3. **基于时序差分**：
   - 可见性检查（visibility check）：地图中有而当前帧没有 = 动态物体移除
   - 帧间差分：连续帧中点消失的位置 = 运动物体

4. **基于Ray-casting**：
   - 从传感器位置做射线投射，被遮挡的点 = 可能来自动态物体

**离群点滤除算法：**

1. **Statistical Outlier Removal (SOR)**：对每个点计算k近邻平均距离，距离超过全局均值+N×std的点视为离群点
2. **Radius Outlier Removal (ROR)**：以每个点为球心，球内邻点数少于阈值的视为离群点
3. **距离阈值过滤**：对应点距离超过阈值直接丢弃
4. **中值距离阈值**：保留d < k · median(d) 的对应
5. **法向量一致性过滤**：对应点法向量差异过大的排除
6. **RANSAC**：随机采样 + 内点共识投票
7. **Trimmed ICP**：只使用最佳匹配的τ比例的点

**参考：**
- [PCL Filters](https://pcl.readthedocs.io/projects/tutorials/en/latest/#filtering)
- [Remove then Revert - Dynamic Object Removal](https://arxiv.org/abs/1810.08596)

---

### 8. 是否熟悉点云去噪、滤波、分割和特征提取等操作和算法？

**解答：**

**点云去噪（Denoising）：**
- **Statistical Outlier Removal**：基于k近邻距离的统计
- **Radius Outlier Removal**：基于球形邻域的点数阈值
- **Bilateral Filter**：同时考虑空间距离和法向量/强度差异
- **MLS (Moving Least Squares)**：局部多项式曲面拟合平滑

**点云滤波（Downsampling）：**
- **Voxel Grid Filter**：体素内取重心 → 最常用
- **Approximate Voxel Grid Filter**：体素内取几何中心 → 更快
- **Uniform Sampling**：等间隔采样
- **Random Sampling**：随机下采样

**点云分割（Segmentation）：**
- **RANSAC平面分割**：提取地面
- **Euclidean Cluster Extraction**：欧式聚类
- **Region Growing**：基于法向量/曲率的区域生长
- **Min-Cut Based Segmentation**：图割分割
- **深度学习方法**：PointNet++, Cylinder3D, KPConv

**特征提取（Feature Extraction）：**
- **PFH / FPFH (Fast Point Feature Histogram)**：描述局部几何，常用于粗配准
- **SHOT (Signature of Histograms of OrienTations)**：局部参考坐标系 + 直方图
- **ISS (Intrinsic Shape Signatures)**：关键点检测
- **NARF (Normal Aligned Radial Feature)**：适合深度图
- **3D-SIFT / 3D-Harris**：关键点检测

**参考：**
- [PCL Tutorials 完整教程](https://pcl.readthedocs.io/projects/tutorials/en/latest/)
- [FPFH (Rusu et al., 2009)](https://ieeexplore.ieee.org/document/5152473)

---

## 六、常见激光SLAM算法

### 1. AMCL、Hector、Karto和Cartographer算法的流程？差异点和贡献点？

**解答：**

**AMCL (Adaptive Monte Carlo Localization)：**
- **类型**：基于粒子滤波的2D定位（非SLAM）
- **流程**：粒子集表示位姿分布 → 运动模型更新粒子 → 用激光scan与已知地图匹配更新权重 → 重采样 → 自适应调整粒子数
- **特点**：需要已知地图，纯定位；KLD采样自适应粒子数

**Hector SLAM：**
- **类型**：基于scan-matching的2D SLAM（无里程计）
- **流程**：高斯-牛顿法优化当前scan与已有地图的对齐 → 多分辨率地图加速 → 增量式构建占据栅格地图
- **贡献**：不需要里程计，只用激光做SLAM；多分辨率匹配
- **缺点**：对初值要求高，快速旋转时容易丢失

**Karto SLAM：**
- **类型**：基于图优化的2D SLAM
- **流程**：前端scan-matching（SPA - Sparse Pose Adjustment）→ 后端图优化（回环检测 + 全局优化）
- **贡献**：较早引入pose graph概念的2D SLAM方案；SPA方法

**Cartographer (Google)：**
- **类型**：基于图优化的2D/3D SLAM
- **流程**：
  - **Local SLAM**：用IMU+里程计初值 → 相关扫描匹配（CSM / Real-Time CSM）→ 生成submap
  - **Global SLAM**：回环检测（基于分支定界的scan-to-submap匹配）→ 位姿图优化
- **贡献点**：
  - 引入了**分支定界（Branch-and-Bound）**加速回环检测
  - Submap机制（多个连续scan构成submap，减少全局优化节点数）
  - 工业级开源方案，工程化程度高

**差异总结：**

| 算法 | 类型 | 核心匹配 | 后端 | 是否需要里程计 |
|------|------|---------|------|---------------|
| AMCL | 纯定位 | 粒子滤波 | 无 | 需要 |
| Hector | SLAM | 高斯-牛顿 | 无回环 | 不需要 |
| Karto | SLAM | SPA | 位姿图 | 需要 |
| Cartographer | SLAM | CSM+BnB | 位姿图 | 最好有 |

**参考：**
- [Cartographer 官方文档](https://google-cartographer.readthedocs.io/)
- [Real-Time Correlative Scan Matching (Olson, 2009)](https://ieeexplore.ieee.org/document/5152473)
- [Hector SLAM](https://wiki.ros.org/hector_slam)

---

### 2. LOAM、LIO-SAM、FAST-LIO算法的流程？差异点和贡献点？

**解答：**

**LOAM (Lidar Odometry and Mapping)：**
- **类型**：纯LiDAR里程计与建图
- **流程**：
  - **特征提取**：根据曲率提取Edge点（大曲率）和Planar点（小曲率）
  - **Lidar Odometry（高频10Hz）**：Edge point-to-edge line + Planar point-to-planar patch配准
  - **Lidar Mapping（低频1Hz）**：当前scan与累积地图配准，更精确
- **贡献**：开创性地提出了两阶段（高频里程计+低频建图）的架构；Edge+Planar特征分离配准
- **缺点**：无IMU，纯激光；无回环

**LIO-SAM (Tightly-coupled Lidar Inertial Odometry via Smoothing and Mapping)：**
- **类型**：紧耦合LiDAR-Inertial SLAM（图优化）
- **流程**：
  - **IMU预积分**：提供高频运动先验 → 去畸变
  - **特征提取**：类似LOAM的Edge+Planar特征
  - **因子图优化**：IMU预积分因子 + LiDAR里程计因子 + GPS因子（可选）+ 回环因子
  - **GTSAM iSAM2**：增量式平滑
- **贡献**：
  - 紧耦合IMU+LiDAR+GPS+回环在一个因子图
  - 使用关键帧机制减少计算量
  - 回环检测用scan-to-map的最邻近搜索
- **特点**：精度高；需要可靠的IMU姿态、外参与噪声参数。工程配置可使用IMU姿态信息初始化，但不能笼统理解为算法理论上必须依赖磁力计。

**FAST-LIO (Fast LiDAR-Inertial Odometry)：**
- **类型**：紧耦合LiDAR-Inertial里程计（滤波/IESKF）
- **流程**：
  - **前向传播**：IMU数据高速预测状态
  - **反向传播**：用IMU积分进行点云运动畸变补偿
  - **IESKF更新**：迭代误差状态卡尔曼滤波，用点到平面距离作为观测
  - **IESKF更新**：在每次迭代中重新计算点面残差和Jacobian，再利用等价的卡尔曼形式高效求解低维状态增量
- **贡献**：
  - 提出公式 $K = (H^T H + P^{-1})^{-1} H^T$ 的等效高效计算 → 计算量与状态维度有关而与观测数无关
  - **IKFoM (Inverse Kalman Filter on Manifold)** 理论框架
  - 极高效率，可处理激光点云的所有原始点
- **FAST-LIO2 改进**：用 **ikd-Tree** 替代KD-Tree，支持增量更新和动态再平衡，进一步加速

**差异对比：**

| 特性 | LOAM | LIO-SAM | FAST-LIO(2) |
|------|------|---------|------------|
| 传感器 | LiDAR | LiDAR+IMU+GPS | LiDAR+IMU |
| 后端 | 无优化 | 因子图(iSAM2) | IESKF |
| 特征 | Edge+Planar | Edge+Planar | 原始点(direct method) |
| 回环 | 无 | 有 | 无(里程计) |
| 计算效率 | 高 | 中 | 极高 |
| 代码质量 | 可读性差 | 好 | 好 |
| 地图结构 | 特征点 | 特征点 | ikd-Tree |

**参考：**
- [LOAM (Zhang & Singh, 2014)](https://www.ri.cmu.edu/pub_files/2014/7/Ji_LidarMapping_RSS2014_v8.pdf)
- [LIO-SAM (Shan et al., 2020)](https://arxiv.org/abs/2007.00258)
- [FAST-LIO2 (Xu et al., 2021)](https://arxiv.org/abs/2107.06829)

---

### 3. 熟悉哪些回环检测算法？影响回环检测精度、鲁棒性的因素有哪些？

**解答：**

**回环检测算法：**

**基于Scan Context的方法：**
- **Scan Context / Scan Context++**：将3D点云投影到2D环形图，用行-列编码描述子做回环检测
- **Stable Triangle Descriptor (STD)**：基于稳定三角形的几何哈希

**基于局部特征的方法：**
- **M2DP**：将点云投影到多个平面做SVD降维
- **LiDAR Iris**：将点云投影成"虹膜图像"用LoG-Gabor滤波+傅里叶变换

**基于直方图的方法：**
- **ESF (Ensemble of Shape Functions)**：角度、面积、距离分布的直方图

**基于深度学习的方法：**
- **OverlapNet / OverlapNet++**：用神经网络预测点云重叠率
- **LCDNet**：3D点云回环检测专用网络
- **PointNetVLAD**：基于PointNet + NetVLAD的检索

**传统几何方法：**
- 位姿图距离阈值 + 配准验证
- RTAB-Map的词袋模型（BoW with LiDAR）

**影响因素：**

1. **视角变化**：同一位置反向经过时点云完全不同（反向回环）
2. **环境变化**：季节变化、动态物体、施工区域
3. **传感器特性**：不同LiDAR型号的分辨率、视场角差异
4. **相似场景**：长走廊不同位置看起来相似 → 感知混淆
5. **描述子设计**：描述子的信息量和不变性（旋转不变性、平移不变性）
6. **搜索策略**：暴力搜索 vs KD-Tree vs 位置先验（缩小搜索半径）

**参考：**
- [Scan Context (Kim & Kim, 2018)](https://ieeexplore.ieee.org/document/8593953)
- [OverlapNet (Chen et al., 2020)](https://arxiv.org/abs/2011.00958)
- [A Survey on Loop Closure Detection for LiDAR SLAM](https://arxiv.org/abs/2208.08566)

---

### 4. 你怎么看待scan2scan、scan2map和map2map的配准方式？你倾向使用哪种？

**解答：**

**scan-to-scan（帧间配准）：**
- **做法**：当前帧与上一帧直接配准
- **优点**：速度快，计算量小
- **缺点**：累积误差大（每步误差累积）；帧间变换小时容易欠约束
- **适用**：运动预测、快速初值估计

**scan-to-map（帧到地图配准）：**
- **做法**：当前帧与历史累积的局部/全局地图配准
- **优点**：精度高（地图包含更多信息，匹配更稳定）；累积误差小
- **缺点**：地图维护开销；地图错误会传播
- **适用**：精确位姿估计（LOAM的Lidar Mapping阶段用的就是scan-to-map）

**map-to-map（地图间配准）：**
- **做法**：两个submap（或多个关键帧的累积）之间配准
- **优点**：信息量最大，最稳定，适合回环验证
- **缺点**：计算量大；需要维护多个submap
- **适用**：回环检测验证、全局重定位、多机器人地图融合

**倾向选择：**
实际工程中常组合使用：**scan-to-scan做快速初值 → scan-to-map做精确估计 → map-to-map做回环验证**。

最常见的高精度方案是**scan-to-map**（如LOAM的mapping阶段），因为它在计算量和精度之间取得了最好的平衡。

**参考：**
- [LOAM 论文](https://www.ri.cmu.edu/pub_files/2014/7/Ji_LidarMapping_RSS2014_v8.pdf)
- [A Benchmark of LiDAR-based SLAM](https://arxiv.org/abs/2007.00258)

---

### 5. 简述占据栅格地图生成的原理？

**解答：**

占据栅格地图（Occupancy Grid Map）是用栅格来表示环境中每个位置是否被占据的概率地图。

**核心原理：**

每个栅格存储一个**占据概率** $p(m_i | z_{1:t}, x_{1:t})$，用**对数几率（log-odds）**表示以简化更新：
$$l_t(m_i) = \log \frac{p(m_i | z_{1:t}, x_{1:t})}{1 - p(m_i | z_{1:t}, x_{1:t})}$$

**更新公式（Binary Bayes Filter）：**
$$l_t(m_i) = l_{t-1}(m_i) + \log \frac{p(m_i | z_t, x_t)}{1 - p(m_i | z_t, x_t)} - \log \frac{p(m_i)}{1 - p(m_i)}$$

即：**当前log-odds = 上次log-odds + 逆传感器模型值 - 先验**

**生成步骤：**

1. 初始化所有栅格为 $l_0 = 0$（即 $p = 0.5$，未知）
2. 对于每个激光点：
   - 从传感器位置到击中点之间做**射线投射（Ray Casting）**
   - 击中点所在栅格 → log-odds增加（`+l_occ`）
   - 射线穿过的栅格 → log-odds减少（`+l_free`）
3. 多个scan累积更新后，即可得到完整的占据栅格地图

**关键参数：**
- $l_{occ}$：击中增加的概率值（如 +0.85）
- $l_{free}$：穿过减少的概率值（如 -0.4）
- 分辨率：栅格大小（典型值5cm/10cm/20cm）

**参考：**
- [Occupancy Grid Mapping (Thrun et al., Probabilistic Robotics)](http://www.probabilistic-robotics.org/)
- [OctoMap - 3D占据栅格](https://octomap.github.io/)

---

### 6. 激光雷达的重复扫描精度在±2cm左右，但SLAM建图精度可达±1cm内，你认为矛盾吗？

**解答：**

**不矛盾。** 这涉及到传感器单次测量精度与估计精度的区别。

**解释：**

1. **统计融合效应**：如果单次测量误差主要是零均值、相互独立的随机噪声，多次观测融合后标准差可近似按 $\sigma / \sqrt{N}$ 下降。但系统误差、标定误差、入射角误差和时间同步误差不会通过简单平均消失，因此不能仅凭重复观测次数保证达到±1cm。

2. **优化/滤波的平滑效应**：SLAM的后端优化（因子图优化 / KF）本质上是在对所有观测进行**加权最小二乘**拟合，这不只是简单的平均，而是利用了**所有传感器数据之间的空间约束关系**来进一步降低不确定性。
   - 例如：$n$ 个位姿由 $m$ 个路标约束，$m > n$ → 超定系统 → 最小二乘解比单次测量更精准

3. **地图表示与内插**：栅格地图/TSDF等隐式表示在融合多帧时可以产生**亚体素精度**的效果。TSDF通过三线性插值可以估计零曲面在体素内部的精确位置。

4. **重复扫描精度 vs 绝对精度**：
   - 重复扫描精度 ±2cm：同一传感器在被测物体不变时，两次测量的差异范围
   - SLAM建图精度 ±1cm：多帧融合后的整体估计误差
   - 两者衡量的是不同的概念

**类比**：GPS单点定位精度米级，但RTK可以达到厘米级——同样的道理。

**参考：**
- [Probabilistic Robotics (Thrun et al.) - 第9章 占据栅格地图](http://www.probabilistic-robotics.org/)
- [Voxblox: Incremental 3D Euclidean Signed Distance Fields](https://arxiv.org/abs/1611.03631)

---

### 7. 熟悉哪些基于LiDAR数据的三维重建算法？请简述Gaussian Splatting的原理？

**解答：**

**基于LiDAR的三维重建算法：**

- **TSDF/ESDF**：Truncated Signed Distance Field，隐式表面表示（如Voxblox, FIESTA）
- **Surfel/Surfel-based**：面片表示（如SuMa++, ElasticFusion）
- **Poisson Surface Reconstruction**：基于法向量的泊松方程求解
- **Mesh Reconstruction**：Delaunay三角化 / Greedy Projection Triangulation
- **NeRF系列**：NeRF, Instant-NGP的LiDAR适配版
- **Gaussian Splatting**：3D高斯泼溅

**3D Gaussian Splatting (3DGS) 原理：**

1. **场景表示**：场景用大量3D高斯椭球（Gaussian Primitives）表示，每个高斯由以下参数定义：
   - 中心位置 $\mu \in \mathbb{R}^3$
   - 协方差矩阵 $\Sigma$（用缩放 $S$ + 旋转 $R$ 分解保证半正定）
   - 颜色（球谐函数系数 SH）或不透明度 $\alpha$

2. **渲染（Splatting/光栅化）**：
   - 将3D高斯投影到2D图像平面
   - 基于深度的alpha blending（类似NeRF体积渲染但高效得多）
   - 用基于tile的排序光栅化器（CUDA实现）

3. **优化**：
   - 损失：L1 + SSIM 渲染损失
   - 自适应密度控制：在梯度大的区域分裂/克隆高斯，在低不透明度区域剔除
   - 从SfM点云初始化

4. **GS-SLAM**：
   - 用SLAM位姿+深度图（LiDAR/RGB-D）作为监督
   - 增量式添加/更新高斯
   - 代表工作：SplaTAM, Gaussian-SLAM, GS-SLAM

**参考：**
- [3D Gaussian Splatting for Real-Time Radiance Field Rendering (Kerbl et al., 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)
- [GS-SLAM: Dense Visual SLAM with 3D Gaussian Splatting](https://arxiv.org/abs/2311.11700)
- [SplaTAM (Keetha et al., 2024)](https://arxiv.org/abs/2312.02126)

---

### 8. 你认为影响GS-SLAM质量的因素有哪些？有什么办法规避？

**解答：**

**影响因素：**

**1. 初始化质量：**
- GS严重依赖好的初始点云（通常来自SfM或SLAM）
- 初始点云稀疏/有噪声 → 高斯分布不合理 → 重建质量差
- **规避**：用更精确的SLAM前端（如LiDAR-SLAM）提供初始点云；多传感器融合

**2. 位姿精度：**
- GS-SLAM是位姿-高斯联合优化（或交替优化）
- SLAM位姿漂移 → 高斯位置偏移 → 不可逆的劣化
- **规避**：紧耦合优化；引入回环检测；分阶段优化（先定姿后调图）

**3. 训练速度和实时性：**
- GS原版渲染快但优化慢（梯度传播到所有高斯）
- SLAM系统需要在实时约束下运行
- **规避**：关键帧机制（不全量训练）；高斯修剪和压缩；增量式更新

**4. 动态场景：**
- 动态物体被建模为永久高斯 → "鬼影"伪影
- **规避**：语义分割剔除动态物体；对运动物体建立短期高斯

**5. 密度控制策略：**
- 自适应密度控制（分裂/克隆/剔除）的参数对场景敏感
- 过密 → 计算量大 + 过拟合；过疏 → 重建缺失
- **规避**：场景自适应的密度阈值；基于LiDAR强度的密度加权

**6. 几何一致性：**
- GS是外观驱动的，几何质量可能不如专门的重建方法
- "浮空"高斯（floaters）→ 假表面
- **规避**：引入深度监督（LiDAR提供真深度）；法向量正则化

**参考：**
- [Gaussian-SLAM (Yugay et al., 2024)](https://arxiv.org/abs/2312.06741)
- [Photo-SLAM: Real-time Monocular SLAM with 3DGS](https://arxiv.org/abs/2312.09075)

---


## 七、3D Gaussian Splatting与神经重建面试题

### 1. 3DGS如何表示一个场景？【高频】

每个高斯包含：

- 三维中心 $\mu$；
- 旋转 $R$ 和尺度 $s$，构成协方差 $\Sigma=RSS^TR^T$；
- 不透明度 $\alpha$；
- 颜色或球谐系数；
- 可选语义、动态概率、时间参数等。

与NeRF隐式MLP不同，3DGS是显式点基元表示，可直接投影和光栅化，训练与渲染速度更高。

---

### 2. 3D高斯如何投影成2D椭圆？

在相机坐标下，将三维均值投影到像素平面；用投影函数在均值处的Jacobian $J$ 近似传播协方差：

$$\Sigma_{2D}\approx J\Sigma_{3D}J^T$$

再结合相机内参得到屏幕空间椭圆。该近似在高斯尺度相对深度较小时较合理。

---

### 3. Alpha blending的顺序为什么重要？

前向渲染按深度从近到远累积：

$$C=\sum_i T_i\alpha_i c_i,\qquad T_i=\prod_{j<i}(1-\alpha_j)$$

前面高斯会遮挡后面高斯，因此必须进行近似深度排序或tile内排序。排序错误会导致颜色和梯度错误，尤其在薄结构和遮挡边界处明显。

---

### 4. 3DGS的densification和pruning分别解决什么？

- Densification：在高梯度、欠拟合或覆盖不足区域克隆/分裂高斯；
- Pruning：删除低透明度、过大、异常或长期不可见的高斯；
- Opacity reset：重新调整透明度分布，防止过早饱和，但使用不当会造成训练抖动或结构坍塌。

核心目标是在表示容量、渲染质量和显存之间取得平衡。

---

### 5. 为什么3DGS初始化点云很重要？

初始化决定高斯从哪里开始覆盖场景。稀疏COLMAP点云可能在低纹理区域缺点；单帧深度直接反投影可能产生多层表面和重复点。

好的初始化应满足：

- 相机内外参和深度尺度一致；
- 多视角反投影后同一表面能重合；
- 进行可见性、深度重投影和体素过滤；
- 点密度与场景尺度匹配；
- 动态区域不污染静态地图。

---

### 6. RGB-D深度是真值，为什么多帧合并后仍可能不适合3DGS？

单帧深度在本帧坐标系正确，不代表多帧反投影后天然形成适合优化的多视角表面采样。问题可能来自：

- 位姿、内参与深度使用约定不一致；
- 深度是z-depth还是ray depth；
- 坐标系左右手、相机到世界/世界到相机搞反；
- 每帧独立采样造成同一表面多层高斯；
- 遮挡边界和深度不连续区域重复；
- 未做跨视角深度一致性筛选。

应通过重投影检查、体素融合、TSDF/surfel融合或多视角置信度筛选构建初始化。

---

### 7. GS-SLAM中跟踪和建图通常如何耦合？

跟踪阶段固定或弱更新地图，通过渲染图像/深度与当前观测比较，优化相机位姿；建图阶段固定或联合优化相机，更新高斯位置、尺度、颜色和透明度。

完全联合优化精度上限高但容易不稳定；工程上常使用交替优化、关键帧、局部地图和不同学习率。

---

### 8. 3DGS-SLAM常见跟踪残差有哪些？

- RGB光度残差；
- 深度残差；
- 轮廓/透明度残差；
- 法向残差；
- 特征重投影残差；
- IMU或外部里程计先验；
- 动态区域掩码和鲁棒权重。

光度残差需要处理曝光、白平衡和动态物体；深度残差要处理无效深度、边缘和尺度。

---

### 9. 如何提高GS重建质量？

从优先级上考虑：

1. 相机位姿、内参和尺度正确；
2. 初始化覆盖和多视角一致性；
3. 关键帧视角分布；
4. 分辨率和采样策略；
5. densification/pruning阈值；
6. 透明度、尺度和球谐学习率；
7. 深度、法向或几何正则；
8. 动态区域与曝光建模。

如果相机几何不自洽，增加复杂几何正则通常治标不治本。

---

### 10. 为什么稀疏视角下“高斯之间的几何约束”可能无效？

在少视角条件下，高斯本身可能位置错误、密度不均、邻域关系不可靠。基于KNN的共面、法向、平滑等约束会把错误邻居强行拉在一起，导致过平滑或局部坍塌。

更稳妥的方向包括：图像/深度先验、伪视角生成、置信度加权、按视线和可见性构建邻域、先提升初始化覆盖再加局部正则。

---

### 11. 如何加速3DGS训练和渲染？

- tile-based rasterization和可见性裁剪；
- 混合精度；
- 限制活跃高斯和局部地图；
- 低分辨率到高分辨率课程训练；
- 减少球谐阶数或延迟开启高阶SH；
- 更高效的densification周期；
- 关键帧采样和误差驱动采样；
- 多GPU或分块地图；
- 对静态背景和动态对象使用不同更新频率。

---

### 12. 如何评价3DGS/GS-SLAM？

渲染指标：PSNR、SSIM、LPIPS；几何指标：深度误差、Chamfer Distance、F-score、法向一致性；轨迹指标：ATE、RPE；系统指标：FPS、训练时间、峰值显存、地图大小、跟踪成功率。

必须明确测试视角是否参与训练、是否使用GT pose/depth、图像是否裁剪、指标区域是否排除无效像素。

---

## 八、SLAM系统对比与架构设计

### 1. Odometry、Localization和SLAM有什么区别？【高频】

- Odometry：估计相邻时刻运动，允许累计漂移；
- Localization：在已知地图中估计当前位姿；
- SLAM：未知环境中同时估计轨迹和地图，通常还要处理回环和全局一致性。

一个系统叫“SLAM”并不代表一定有完整回环；面试时要具体说明地图是否长期维护、是否支持重定位和全局优化。

---

### 2. 前端和后端分别负责什么？

前端：数据预处理、特征/几何提取、匹配、初始位姿、异常值剔除；后端：状态建模、滤波或优化、边缘化、回环和地图维护。

前端决定观测质量，后端决定如何融合信息。很多“后端发散”实际根因是前端错误对应、时间同步或标定。

---

### 3. 滤波式和优化式系统如何选择？【高频】

- ESKF/MSCKF：低延迟、协方差天然、计算更可预测，适合嵌入式和控制；
- 滑窗优化：可多次重线性化、扩展因子方便、精度上限高；
- 全局图优化：适合回环和长期一致性；
- 混合架构：高频滤波/传播输出，低频优化器校正。

不能笼统说“优化一定更准”或“滤波一定更实时”，实现、状态规模和传感器配置同样重要。

---

### 4. ORB-SLAM3、OpenVINS、VINS-Mono和Basalt的核心差异？

**标准回答：** ORB-SLAM3是完整优化式SLAM；VINS-Mono是滑窗优化式单目VIO加4DoF位姿图；OpenVINS是ESKF/MSCKF滤波式VIO；Basalt是使用平方根边缘化与QR消点的固定时滞优化式VIO。

进一步比较前端：ORB描述子适合回环和宽基线；VINS/OpenVINS多用KLT连续跟踪；Basalt使用patch optical flow。

---

### 5. 松耦合、紧耦合和深耦合怎么区分？

- 松耦合：融合各子系统最终位姿/速度结果；
- 紧耦合：在统一状态估计器中使用各传感器原始或中间观测残差；
- “深耦合”常用于表示模型或网络层级更深的联合，但没有完全统一的数学定义，回答时应明确具体耦合变量和信息流。

紧耦合通常信息利用率高，但开发、标定和故障隔离更复杂。

---

### 6. 如何设计SLAM多线程架构？

常见线程：传感器接收、前端跟踪、局部建图、回环检测、全局优化、可视化和日志。

关键问题：

- 数据时间顺序和队列容量；
- 地图读写锁和快照；
- 回环优化后位姿修正如何通知跟踪线程；
- 线程退出和异常处理；
- 避免持锁执行耗时优化；
- 对实时线程设置优先级和背压策略。

---

### 7. 地图应该选点云、体素、TSDF、Surfel还是高斯？

- 稀疏点：定位和回环高效；
- 稠密点云：直观但冗余；
- Occupancy/Octree：导航和碰撞；
- TSDF：表面融合和Mesh；
- Surfel：增量表面；
- 3DGS：高质量新视角渲染；
- NeRF：连续隐式场，但在线优化成本高。

地图类型由下游任务决定，不存在单一最优表示。

---

### 8. 如何设计系统的失败恢复？

- 跟踪质量评分：内点数、残差、协方差、退化特征值；
- 短时失败：IMU/轮速传播；
- 中期失败：局部重定位；
- 长期失败：创建新子地图；
- 找回旧区域：地图融合；
- 保护机制：冻结地图更新，避免错误位姿污染地图。

---

### 9. 如何处理实时性与精度冲突？

可采用：固定窗口、关键帧、局部地图、多分辨率、分层频率、异步回环、动态特征数量、早停、GPU加速、地图分块和按负载降级。

面试最好给出量化指标：前端多少毫秒、后端多少毫秒、最坏延迟、峰值内存，而不是只说“实时”。

---

### 10. 从零设计一个视觉惯性SLAM系统，你会如何分模块？

1. 时间同步和标定；
2. 图像/IMU预处理；
3. 特征跟踪与异常剔除；
4. 初始化；
5. IMU预积分和状态传播；
6. 滑窗滤波/优化；
7. 关键帧和地图管理；
8. 回环候选与几何验证；
9. 位姿图/全局优化；
10. 重定位和多地图；
11. 评测、日志、可视化和故障恢复。

回答时应说明每个模块输入输出、线程关系、状态定义和失败模式。



## 九、C++基础问题

### 1. 面向过程和面向对象编程的区别？优缺点？

**解答：**

**面向过程（Procedural Programming）：**
- 以**函数**为核心，程序 = 数据结构 + 函数
- 强调"做什么"的步骤/流程
- 代表：C语言

**面向对象（Object-Oriented Programming, OOP）：**
- 以**对象**为核心，对象 = 数据 + 方法
- 四大特性：**封装、继承、多态、抽象**
- 代表：C++, Java, Python

| 维度 | 面向过程 | 面向对象 |
|------|---------|---------|
| 核心 | 函数 | 对象（类） |
| 优点 | 简单直接；执行效率高；内存可控 | 易维护/扩展；代码复用（继承）；模块化好 |
| 缺点 | 扩展性差；代码复用难；大项目维护难 | 性能开销（虚函数表）；设计过度可能增加复杂度 |
| 适用 | 底层系统、嵌入式、算法 | 大规模应用、GUI、框架 |

**参考：**
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)

---

### 2. 指针和引用的区别？

**解答：**

| 特性 | 指针 (Pointer) | 引用 (Reference) |
|------|---------------|-----------------|
| 定义 | `int *p = &a;` | `int &r = a;` |
| 是否可为空 | 可以 `nullptr` | 不可以（必须绑定到合法对象） |
| 可否重新绑定 | 可以 `p = &b;` | 不可以（一经绑定不可更改） |
| 可否做算术运算 | 可以 `p++` | 不可以 |
|  sizeof  | 指针自身大小(8/4字节) | 引用对象的大小 |
| 使用方式 | `*p` 解引用 | 直接使用 `r` |
| 多级 | 可以有二级指针 `int **pp` | 无多级引用 |
| 底层实现 | 存储地址 | 编译期就是别名（底层也是地址） |

**面试关键点**：引用更安全（不可为空），指针更灵活（可为空、可重新指向、可算术运算）。在函数参数中使用 `const &` 可以避免拷贝且保证不修改原值。

---

### 3. 什么是多态？虚函数和纯虚函数的区别和作用？

**解答：**

**多态（Polymorphism）：**
- **编译时多态**（静态多态）：函数重载、运算符重载、模板
- **运行时多态**（动态多态）：通过**虚函数**实现，基类指针/引用调用派生类方法

**虚函数（virtual function）：**
- 基类中声明 `virtual`，派生类可以重写（override）
- 通过虚函数表（vtable）实现动态绑定
- 基类必须有实现
- 可以有派生类不重写的默认行为

**纯虚函数（pure virtual function）：**
- 声明为 `virtual void func() = 0;`，基类**不需要实现**
- 包含纯虚函数的类是**抽象类**，不能实例化
- 强制所有派生类必须实现此函数（否则派生类也是抽象类）
- 用于定义**接口**

```cpp
class Base {
public:
    virtual void func1() { /* 默认实现 */ }    // 虚函数
    virtual void func2() = 0;                   // 纯虚函数
};
```

| 特性 | 虚函数 | 纯虚函数 |
|------|--------|---------|
| 基类实现 | 必须有 | 不能有（=0） |
| 派生类重写 | 可选 | 强制 |
| 抽象类 | 不产生抽象类 | 产生抽象类 |
| 作用 | 提供可重写的默认行为 | 定义接口契约 |

**参考：**
- [C++ virtual functions](https://en.cppreference.com/w/cpp/language/virtual)

---

### 4. 为什么基类的析构函数建议定义为虚函数？

**解答：**

如果基类析构函数不是虚函数，通过**基类指针删除派生类对象**时，只会调用基类的析构函数，不会调用派生类的析构函数 → **派生类资源泄漏**。

```cpp
class Base {
public:
    ~Base() { /* 非虚析构 */ }   // ❌ 危险的
};

class Derived : public Base {
    int* data;
public:
    Derived() : data(new int[100]) {}
    ~Derived() { delete[] data; }
};

Base* p = new Derived();
delete p;  // 只调用Base::~Base()！Derived的data泄漏了！
```

**正确做法：**
```cpp
class Base {
public:
    virtual ~Base() = default;  // ✅ 虚析构
};
```

**C++标准规定**：带虚函数的类应该声明虚析构函数。这是为了确保析构时从派生类到基类的完整析构链被正确调用。

**例外：** 如果确定类不会被继承，不需要虚析构（加 `final` 明确标记）。

---

### 5. C++提供的类型转换有几种？各自的特点？

**解答：**

C++提供4种命名的类型转换：

| 转换 | 用途 | 特点 |
|------|------|------|
| **`static_cast`** | 编译时检查的显式转换 | 相关类型间转换（int→float, 基类→派生类【不安全】）；不检查运行时；最常用 |
| **`dynamic_cast`** | 运行时检查的安全向下转换 | 仅用于多态类型（有虚函数）；失败返回nullptr(指针)或抛异常(引用)；有性能开销 |
| **`const_cast`** | 添加/移除const/volatile | 唯一能改变cv限定符的转换；**不改变值** |
| **`reinterpret_cast`** | 位级别的任意类型重解释 | 最危险；不改变二进制表示；极度依赖平台和编译器 |

**C风格转换 `(type)expr`**：以上四种任意组合尝试，不安全，现代C++应避免。

**参考：**
- [C++ Casting Operators](https://en.cppreference.com/w/cpp/language/explicit_cast)

---

### 6. 通过基类指针/引用调用派生类对象时，应该用哪种类型转换？

**解答：**

用 **`dynamic_cast`**。

原因：`dynamic_cast` 在运行时会检查类型是否匹配，如果转换失败：
- 对指针：返回 `nullptr`
- 对引用：抛出 `std::bad_cast` 异常

这保证了**类型安全**。

```cpp
Base* base = new Derived();
Derived* derived = dynamic_cast<Derived*>(base);
if (derived) {
    // 安全使用
}
```

`static_cast` 也能完成基类→派生类的转换，但**不做运行时检查**，如果实际类型不对 → 未定义行为。

---

### 7. dynamic_cast和static_cast，哪个运行时效率高？为什么？

**解答：**

**`static_cast` 效率更高。**

**原因：**
- `static_cast` 是**编译时转换**，转换操作在编译期就确定了，运行时就是直接使用转换后的类型（零运行时开销）
- `dynamic_cast` 是**运行时转换**，需要遍历虚函数表中的RTTI（运行时类型信息），检查类继承关系，开销显著（特别在深层继承链中）

**实际数据：** `dynamic_cast` 的开销可能是 `static_cast` 的10-100倍，具体取决于继承深度和编译器实现。

**工程建议：**
- 如果确定类型（比如从容器取出时知道确切类型）→ 用 `static_cast`
- 如果不确定类型且需要安全检查 → 用 `dynamic_cast`
- 尽量避免在热循环中使用 `dynamic_cast`

---

### 8. 智能指针有哪几种？区别是什么？

**解答：**

C++11起提供三种主要智能指针：

| 智能指针 | 所有权 | 拷贝 | 特点 |
|---------|--------|------|------|
| **`unique_ptr`** | 独占所有权 | 不可拷贝，只能移动(move) | 零开销（与裸指针同大小）；默认选择 |
| **`shared_ptr`** | 共享所有权 | 可拷贝（引用计数+1） | 引用计数（线程安全）；控制块开销 |
| **`weak_ptr`** | 不拥有（观察） | 可拷贝 | 配合shared_ptr打破循环引用；`.lock()`临时获取shared_ptr |

**区别总结：**

```cpp
// unique_ptr：独占，move语义
std::unique_ptr<int> p1 = std::make_unique<int>(42);
std::unique_ptr<int> p2 = std::move(p1);  // p1变为nullptr

// shared_ptr：共享，引用计数
std::shared_ptr<int> p3 = std::make_shared<int>(42);
std::shared_ptr<int> p4 = p3;  // 引用计数=2

// weak_ptr：不增加引用计数
std::weak_ptr<int> p5 = p3;
if (auto sp = p5.lock()) { /* 使用sp */ }
```

**选择建议：** 默认 `unique_ptr` → 需要共享所有权时 `shared_ptr` → 打破循环引用/观察时 `weak_ptr`。

**参考：**
- [C++ Smart Pointers](https://en.cppreference.com/w/cpp/memory)

---

### 9. 当程序试图将一个unique_ptr赋值给另一个时，什么情况下编译器允许？

**解答：**

编译器只在以下情况允许：

1. **移动语义（std::move）**：
```cpp
std::unique_ptr<int> p1 = std::make_unique<int>(42);
std::unique_ptr<int> p2 = std::move(p1);  // ✅ 通过移动构造/赋值
```

2. **函数返回（RVO/NRVO，隐式move）**：
```cpp
std::unique_ptr<int> create() {
    auto p = std::make_unique<int>(42);
    return p;  // ✅ 编译器自动move（C++14起）
}
```

3. **临时对象（右值）**：
```cpp
std::unique_ptr<int> p = std::make_unique<int>(42);  // ✅ 右边是临时对象（右值）
```

**不允许的情况：**
```cpp
std::unique_ptr<int> p1 = std::make_unique<int>(42);
std::unique_ptr<int> p2 = p1;  // ❌ 编译错误！拷贝构造被delete
```

核心原因：`unique_ptr` 的拷贝构造函数和拷贝赋值运算符被**显式删除**（`= delete`），只有移动构造/移动赋值是合法的。

---

### 10. C++中提供了哪些锁？各自的作用和适用场景？

**解答：**

| 锁类型 | 特点 | 适用场景 |
|--------|------|---------|
| **`std::mutex`** | 基本互斥锁，不可递归 | 简单互斥保护 |
| **`std::recursive_mutex`** | 同一线程可多次lock | 递归函数中需要锁 |
| **`std::timed_mutex`** | 带超时的互斥锁 | 需要超时机制的加锁 |
| **`std::shared_mutex`** (C++17) | 读写锁：多读单写 | 读多写少场景（如配置读取） |
| **`std::spinlock`** (无标准) | 自旋锁，忙等待 | 临界区极短（几微秒），如中断上下文 |
| **`std::lock_guard`** | RAII自动加解锁 | 简单作用域内加锁 |
| **`std::unique_lock`** | RAII + 可延迟加锁/提前解锁/转移 | 需要灵活控制锁生命周期 |
| **`std::scoped_lock`** (C++17) | RAII + 死锁避免（同时锁多个） | 需要锁多个互斥量 |
| **`std::condition_variable`** | 条件变量（配合mutex） | 生产者-消费者模式 |

```cpp
// 典型用法
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);  // 自动加锁/解锁
    // 临界区
}

// 读写锁：多读单写
std::shared_mutex rw_mtx;
// 读
std::shared_lock lock(rw_mtx);
// 写
std::unique_lock lock(rw_mtx);

// 避免死锁：同时锁两个
std::scoped_lock lock(mtx1, mtx2);
```

**参考：**
- [C++ Concurrency Support](https://en.cppreference.com/w/cpp/thread)

---

### 11. C++中堆与栈从内存上和数据结构上的区分？

**解答：**

**从内存分配角度：**

| 特性 | 栈 (Stack) | 堆 (Heap) |
|------|-----------|----------|
| 管理方式 | 编译器自动管理 | 程序员手动管理（new/delete）或智能指针 |
| 分配速度 | 极快（移动栈指针即可） | 较慢（需要查找空闲块） |
| 大小限制 | 有限（通常1-8MB） | 系统可用内存限制（GB级别） |
| 生命周期 | 作用域结束自动释放 | 手动释放或智能指针析构 |
| 碎片问题 | 无（LIFO） | 可能产生内存碎片 |
| 线程安全 | 天然线程安全（每线程独立） | 需要同步机制 |
| 典型用途 | 局部变量、函数参数 | 动态大小数据、跨函数共享 |

**从数据结构角度：**

| 特性 | 栈 (Stack) | 堆 (Heap / Priority Queue) |
|------|-----------|---------------------------|
| 操作规则 | LIFO（后进先出） | 优先级有序（最大/最小堆） |
| 插入删除 | 只能在栈顶 | 任意位置，但需维护堆序 |
| 典型操作 | push, pop, top | push, pop, top (O(log n)) |
| 实现 | 数组或链表 | 完全二叉树（用数组实现） |
| STL对应 | `std::stack` | `std::priority_queue` |

---

### 12. C++ STL的容器有哪些？各自的特点？

**解答：**

**序列容器（Sequence Containers）：**

| 容器 | 底层结构 | 随机访问 | 头插入 | 尾插入 | 中间插入 | 内存 |
|------|---------|---------|--------|--------|---------|------|
| **`vector`** | 动态数组 | O(1) | O(n) | O(1)* | O(n) | 连续 |
| **`deque`** | 分段数组 | O(1) | O(1) | O(1) | O(n) | 分段连续 |
| **`list`** | 双向链表 | O(n) | O(1) | O(1) | O(1)** | 离散 |
| **`forward_list`** | 单向链表 | O(n) | O(1) | - | O(1)** | 离散 |
| **`array`** | 静态数组 | O(1) | - | - | - | 连续（栈上） |

\* 均摊 \*\* 已有迭代器定位后

**关联容器（Associative Containers）：**

| 容器 | 底层结构 | 查找 | 插入 | 有序 | 重复键 |
|------|---------|------|------|------|--------|
| **`set`** | 红黑树 | O(log n) | O(log n) | ✅ | ❌ |
| **`multiset`** | 红黑树 | O(log n) | O(log n) | ✅ | ✅ |
| **`map`** | 红黑树 | O(log n) | O(log n) | ✅ (key) | ❌ |
| **`multimap`** | 红黑树 | O(log n) | O(log n) | ✅ (key) | ✅ |

**无序关联容器（C++11）：**

| 容器 | 底层结构 | 平均查找 | 最坏查找 | 有序 |
|------|---------|---------|---------|------|
| **`unordered_set`** | 哈希表 | O(1) | O(n) | ❌ |
| **`unordered_map`** | 哈希表 | O(1) | O(n) | ❌ |

**选择建议：**
- 需要随机访问 → `vector`
- 头尾都插入 → `deque`
- 频繁中间插入删除 → `list`
- 有序 + 按键查找 → `map`/`set`
- 无序 + 快速查找 → `unordered_map`/`unordered_set`

---

### 13. vector\<int\>与vector\<bool\>的特性区别？

**解答：**

`vector<bool>` 是C++ STL的一个**特化版本**，与 `vector<int>` 有本质区别：

| 特性 | `vector<int>` | `vector<bool>` |
|------|--------------|----------------|
| 存储方式 | 每个元素占sizeof(int)字节 | **位压缩存储**（每个bool占1 bit） |
| `operator[]` 返回 | `int&` | **代理对象**（proxy reference），不是 `bool&` |
| `data()` | 返回 `int*` | **不存在**（无法返回位级别的指针） |
| 内存占用 | 4 bytes/元素 | 1 bit/元素（省32倍内存） |
| 是否可以取地址 | 可以 `&v[0]` | **不可以** `&v[0]` 编译错误 |

**重要陷阱：**
```cpp
std::vector<bool> v = {true, false, true};
auto x = v[0];     // x 是代理对象，不是bool！
v[0] = false;       // ✅ 可以修改
bool& r = v[0];    // ❌ 编译错误！代理对象不能绑定到bool&
```

**建议：** 如果需要位压缩存储，用 `std::bitset<N>` 或 `boost::dynamic_bitset<>`；如果需要正常的STL行为，用 `std::vector<char>` 或 `std::deque<bool>`。

---

### 14. vector的resize和reserve的区别？

**解答：**

| 特性 | `resize(n)` | `reserve(n)` |
|------|------------|-------------|
| 改变size | ✅ size = n | ❌ size不变 |
| 改变capacity | ✅ capacity ≥ n | ✅ capacity ≥ n |
| 构造/析构元素 | ✅ 新增元素被构造（默认值或指定值） | ❌ 不操作元素 |
| 访问新元素 | ✅ 可以（已被构造） | ❌ 不能（虽然内存已分配但未构造） |
| 常见用途 | 确定需要n个元素 | 预先分配内存避免反复reallocation |

```cpp
std::vector<int> v;
v.reserve(100);    // capacity=100, size=0
v[0];              // ❌ 未定义行为！size=0

v.resize(100);     // capacity=100, size=100
v[0];              // ✅ OK
v.push_back(1);    // size=101
```

**关键区别：** `reserve` 不改变逻辑大小，不构造元素；`resize` 改变逻辑大小并构造元素。

---

### 15. map与unordered_map的区别？底层实现？时间效率选哪个？

**解答：**

| 特性 | `map` | `unordered_map` |
|------|-------|----------------|
| **底层实现** | 红黑树（平衡二叉搜索树） | 哈希表 |
| **有序性** | ✅ 按键排序 | ❌ 无序 |
| **查找** | O(log n) | O(1) 平均，O(n) 最坏 |
| **插入** | O(log n) | O(1) 平均，O(n) 最坏（rehash） |
| **删除** | O(log n) | O(1) 平均 |
| **内存** | 较小（每个节点左右孩子+父指针） | 较大（桶数组+链表/开放定址） |
| **迭代** | 有序遍历 | 遍历顺序依赖哈希桶 |
| **要求** | key 必须支持 `operator<` | key 必须支持 `hash<Key>` 和 `operator==` |

**时间效率选择：**
- **追求时间效率 → `unordered_map`**（绝大多数情况O(1)更快）
- 需要有序遍历 → `map`（如需要按key顺序输出）
- 对内存敏感 → `map`
- 最坏情况不可接受 → `map`（如果哈希冲突攻击是风险）

**Map底层（红黑树）简析：**
- 自平衡二叉搜索树
- 保证树的高度不超过 $2\log(n+1)$
- 插入/删除通过旋转和重新着色维持平衡

**参考：**
- [Hash table vs Red-Black tree performance](https://en.cppreference.com/w/cpp/container)

---

### 16. 如果必须插入到容器的中间，且访问频繁，有什么建议？

**解答：**

**分析：** "插入到中间" → 要求非连续存储，"访问频繁" → 要求快速随机访问或遍历。

**推荐方案：**

1. **如果"访问"是遍历/迭代**：用 `std::list`（双向链表）
   - 中间插入 O(1)（已有迭代器定位）
   - 遍历 O(n)

2. **如果"访问"需要随机访问 + 插入在中间**：这是矛盾的（没有完美方案），需要权衡
   - 数据量不大（<几千）：`std::vector` + 接受 O(n) 的中间插入（内存局部性弥补）
   - 数据量大且随机访问多：考虑**跳表**（如Redis）或 **B+树** 结构

3. **基于哈希的容器变体**：
   - 如果"插入到中间"是指任意位置，用 `unordered_map<int, T>`（key是位置索引）
   - 中间插入不需要移动元素，只需更新索引映射

4. **折中方案**：
   - `std::deque`：分段数组，头尾快，中间比vector快一些（块内不需要移动所有元素）

**实际工程建议：** 大多数情况 `std::vector` 足够了。现代CPU的缓存局部性优势使得 vector 的 O(n) 插入在小数据量下可能比 list 的 O(1) 插入更快。

---

### 17. 怎么理解C++中左值和右值？右值引用的常见使用场景？

**解答：**

**左值（lvalue）**：有标识、可寻址、持久的表达式
- 可以取地址 `&x`
- 出现在赋值左边
- 变量名一般是左值

**右值（rvalue）**：临时的、匿名的、即将销毁的表达式
- 不能取地址（`&` 对右值编译错误）
- 字面量、临时对象、函数返回的非引用值

```cpp
int a = 10;        // a是左值, 10是右值
int b = a + 1;     // b是左值, a+1是右值(临时)
int&& r = a + 1;   // r是右值引用，绑定到临时结果
int&& r2 = a;      // ❌ 编译错误！不能将右值引用绑定到左值
int&& r3 = std::move(a);  // ✅ std::move将左值转为右值
```

**右值引用的使用场景：**

1. **移动语义（Move Semantics）— 最重要**：
```cpp
class MyVector {
    int* data; size_t size;
public:
    // 移动构造：直接"偷"资源，避免深拷贝
    MyVector(MyVector&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }
};
```

2. **完美转发（Perfect Forwarding）**：
```cpp
template<typename T, typename Arg>
std::shared_ptr<T> factory(Arg&& arg) {
    return std::make_shared<T>(std::forward<Arg>(arg));
}
```

3. **避免不必要的拷贝**：
```cpp
void process(std::string&& s) {  // 只接受右值
    data_ = std::move(s);  // 内部move而非copy
}
```

**参考：**
- [Move Semantics and Rvalue References](https://en.cppreference.com/w/cpp/language/move_constructor)
- [Effective Modern C++ (Scott Meyers) - 第5章](https://www.oreilly.com/library/view/effective-modern-c/9781491908412/)

---


## 十、ROS/ROS2与SLAM工程开发

### 1. Topic、Service和Action分别适合什么场景？【高频】

- Topic：连续数据流，如图像、点云、IMU；
- Service：短时请求—响应，如重置地图、保存配置；
- Action：耗时任务，可反馈进度和取消，如导航到目标点、离线建图。

不要用Service持续传图像，也不要用Topic模拟必须确认成功的短事务。

---

### 2. ROS2 QoS有哪些关键策略？

- Reliability：Reliable / Best Effort；
- Durability：Volatile / Transient Local；
- History：Keep Last / Keep All；
- Depth：队列长度；
- Deadline、Lifespan、Liveliness。

传感器高频数据常用Best Effort以避免阻塞；地图或静态配置可能使用Transient Local。发布端和订阅端QoS不兼容会导致“话题存在但收不到数据”。

---

### 3. ROS时间、系统时间和稳态时钟有什么区别？

- ROS Time可由仿真 `/clock` 驱动，支持bag回放；
- System Time可能被NTP校时跳变；
- Steady Clock单调递增，适合测量耗时。

SLAM时间戳应来自传感器采样时刻，而不是回调到达时刻。

---

### 4. 如何实现相机和IMU时间同步？

优先级：硬件触发/PPS > 驱动层统一时钟 > 在线时间偏移估计 > 软件近似同步。

ROS中可使用message_filters的ExactTime或ApproximateTime，但IMU高频积分通常需要按相机时间截取区间并对边界做插值，不能简单选最近一条IMU。

---

### 5. tf2中map、odom和base_link通常如何定义？【高频】

- `map`：全局一致坐标，回环后可跳变；
- `odom`：局部连续坐标，短期平滑但长期漂移；
- `base_link`：机器人机体坐标。

常见树：`map -> odom -> base_link -> sensors`。全局定位更新 `map->odom`，里程计更新 `odom->base_link`。

---

### 6. ROS2 Executor和Callback Group有什么作用？

Executor决定回调调度；Callback Group决定回调是否可并行。

- MutuallyExclusive：组内串行；
- Reentrant：允许并行进入；
- SingleThreadedExecutor：易推理；
- MultiThreadedExecutor：提高吞吐，但需要线程安全。

SLAM地图写操作应避免在多个回调中无锁并发。

---

### 7. 如何避免ROS回调阻塞？

- 回调只做时间戳检查和入队；
- 重计算放到工作线程；
- 设置有界队列和丢帧策略；
- 不在锁内做点云配准或优化；
- 对图像使用共享指针/零拷贝；
- 记录排队延迟和处理延迟。

---

### 8. 什么是ROS2组件化和进程内通信？

Composable Node可将多个节点加载到同一进程，减少序列化和拷贝。配合intra-process communication可提高图像/点云吞吐。

代价是进程隔离变弱，一个组件崩溃可能影响全部组件，调试和线程安全要求更高。

---

### 9. rosbag/rosbag2回放时要注意什么？

- 是否使用仿真时间；
- 回放速率和消息时间戳；
- QoS覆盖；
- 多topic原始时间顺序；
- `/tf_static` 是否完整；
- bag压缩带来的CPU负载；
- 算法是否按消息时间而非墙钟时间超时。

离线评测要保证每次参数、随机种子和播放设置一致。

---

### 10. ROS参数和动态参数如何安全使用？

启动时验证范围、单位和必填项；动态修改时通过回调检查合法性。影响状态维度、坐标系或地图结构的参数通常不应运行中直接修改。

配置文件中应明确单位，例如IMU噪声密度不是离散标准差。

---

### 11. Lifecycle Node适合什么场景？

生命周期节点具有unconfigured、inactive、active、finalized等状态，适合需要严格启动顺序的传感器、定位和导航系统。

可在configure阶段加载地图和标定，在activate阶段开始发布，在deactivate阶段停止处理但保留资源。

---

### 12. 如何调试ROS系统中的延迟和丢帧？

- 查看topic频率、带宽和时间戳差；
- 记录消息生成、接收、入队、出队和处理完成时间；
- 检查CPU占用、调度、DDS队列和QoS；
- 检查图像是否在解码或转换中拷贝多次；
- 使用ros2 tracing、perf、火焰图；
- 将算法处理时间与排队时间分开统计。

---

### 13. catkin、ament和colcon的区别？

ROS1常用catkin；ROS2使用ament作为构建与包规范，colcon负责工作空间级并行构建和测试。现代工程应使用正确的依赖导出、install规则和接口生成方式，避免只在源码目录可运行。

---

### 14. 如何设计自定义消息？

优先复用标准消息。自定义消息应：

- 带Header和采样时间；
- 明确坐标系；
- 字段单位写入注释；
- 避免嵌套过深和超大动态数组；
- 对高频数据考虑内存布局和兼容性；
- 修改接口后做好版本管理。

---

### 15. ROS工程面试中的常见故障题

**问题：topic频率正常，但SLAM轨迹断断续续。怎么排查？**

依次检查：采样时间是否单调、图像与IMU区间是否完整、队列是否丢帧、算法处理是否超过输入周期、tf查询是否超时、CPU调度和锁竞争、传感器驱动是否批量发送旧数据。



## 十一、SLAM工程实践算法题

### 1. 基于ceres/g2o/gtsam写一个自定义edge/factor/residual function

**解答：** 详见上文 [13. 优化库自定义残差](#13-是否熟悉ceresg2o或gtsam等优化库如何自定义构建残差方程)

---

### 2. 基于空间中两个坐标点，写一个高效的碰撞检测函数

**解答：**

```cpp
#include <Eigen/Dense>

// 最简单的碰撞检测：两点距离 < 阈值
// 实际使用中碰撞检测一般用于AABB/OBB/BVH等包围盒结构
bool checkCollision(const Eigen::Vector3d& p1, const Eigen::Vector3d& p2, 
                    double threshold = 0.1) {
    return (p1 - p2).squaredNorm() < threshold * threshold;
}

// 如果是带有半径的球体碰撞检测
bool sphereCollision(const Eigen::Vector3d& c1, double r1,
                     const Eigen::Vector3d& c2, double r2) {
    double dist2 = (c1 - c2).squaredNorm();
    double sum_r = r1 + r2;
    return dist2 < sum_r * sum_r;
}

// 高效版本：使用平方距离避免开根号
struct BoundingSphere {
    Eigen::Vector3d center;
    double radius;
};

bool efficientCollision(const BoundingSphere& a, const BoundingSphere& b) {
    double threshold = a.radius + b.radius;
    return (a.center - b.center).squaredNorm() < threshold * threshold;
}
```

**关键优化：** 用 `squaredNorm()` 而非 `norm()` 避免开根号（`sqrt` 操作）。

---

### 3. 基于给定的空间三角形，写一个在该三角形中生成随机数的函数

**解答：**

```cpp
#include <Eigen/Dense>
#include <random>

// 用重心坐标法在三角形内生成均匀随机点
Eigen::Vector3d randomPointInTriangle(
    const Eigen::Vector3d& A, 
    const Eigen::Vector3d& B, 
    const Eigen::Vector3d& C) {
    
    static std::random_device rd;
    static std::mt19937 gen(rd());
    static std::uniform_real_distribution<double> dis(0.0, 1.0);
    
    double u = dis(gen);
    double v = dis(gen);
    
    // 如果u+v>1，翻转到三角形内（保证均匀分布）
    if (u + v > 1.0) {
        u = 1.0 - u;
        v = 1.0 - v;
    }
    
    // 重心坐标插值
    return (1.0 - u - v) * A + u * B + v * C;
}
```

**原理：** 重心坐标中 $P = \alpha A + \beta B + \gamma C$，其中 $\alpha + \beta + \gamma = 1$ 且 $\alpha, \beta, \gamma \geq 0$。`u+v>1` 的翻转保证了在三角形内均匀分布（否则会在平行四边形内均匀分布）。

---

### 4. 写一个基于状态量pose的插值函数

**解答：**

```cpp
#include <Eigen/Dense>
#include <Sophus/SE3.hpp>

struct TimedPose {
    double time_stamp;
    Sophus::SE3d pose;  // 或者 Eigen::Isometry3d
};

// 基于时间的SE(3)位姿插值
Sophus::SE3d interpolatePose(const TimedPose& tp1, const TimedPose& tp2, double query_time) {
    double t1 = tp1.time_stamp;
    double t2 = tp2.time_stamp;
    
    // 计算插值因子
    double alpha = (query_time - t1) / (t2 - t1);
    alpha = std::clamp(alpha, 0.0, 1.0);
    
    // 方法1：用Sophus的SE3直接插值
    // SE3 = exp(alpha * log(T2 * T1^{-1})) * T1
    Sophus::SE3d relative = tp2.pose * tp1.pose.inverse();
    Eigen::Vector6d relative_se3 = relative.log();  // 到李代数
    
    Sophus::SE3d interpolated = Sophus::SE3d::exp(alpha * relative_se3) * tp1.pose;
    
    return interpolated;
}

// 更高效版本：如果只需要平移插值+旋转插值分开做
Sophus::SE3d interpolatePoseSplit(const TimedPose& tp1, const TimedPose& tp2, double query_time) {
    double alpha = (query_time - tp1.time_stamp) / (tp2.time_stamp - tp1.time_stamp);
    alpha = std::clamp(alpha, 0.0, 1.0);
    
    // 旋转用slerp（球面线性插值）
    Eigen::Quaterniond q1(tp1.pose.rotationMatrix());
    Eigen::Quaterniond q2(tp2.pose.rotationMatrix());
    Eigen::Quaterniond q_interp = q1.slerp(alpha, q2);
    
    // 平移用线性插值
    Eigen::Vector3d t_interp = (1.0 - alpha) * tp1.pose.translation() 
                               + alpha * tp2.pose.translation();
    
    Sophus::SE3d result(q_interp, t_interp);
    return result;
}
```

**参考：**
- [Sophus Library](https://github.com/strasdat/Sophus)
- [Quaternion Slerp](https://en.wikipedia.org/wiki/Slerp)

---

### 5. 给定n个submap，按升序输出相似度

**解答：**

```cpp
#include <vector>
#include <set>
#include <unordered_set>
#include <algorithm>
#include <iostream>

// 计算两个submap的Jaccard相似度
double jaccardSimilarity(const std::vector<int>& submap1, 
                         const std::vector<int>& submap2) {
    std::unordered_set<int> set1(submap1.begin(), submap1.end());
    std::unordered_set<int> set2(submap2.begin(), submap2.end());
    
    // 计算交集大小
    size_t intersection = 0;
    for (int val : set1) {
        if (set2.count(val)) {
            intersection++;
        }
    }
    
    // 并集大小 = |A| + |B| - |A∩B|
    size_t union_size = set1.size() + set2.size() - intersection;
    
    if (union_size == 0) return 0.0;
    return static_cast<double>(intersection) / union_size;
}

// 按相似度升序输出所有submap对
// 使用set自动排序，存储{相似度, i, j}
std::vector<std::tuple<double, int, int>> 
orderedSubmapSimilarities(const std::vector<std::vector<int>>& submaps) {
    std::vector<std::tuple<double, int, int>> result;
    
    int n = submaps.size();
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            double sim = jaccardSimilarity(submaps[i], submaps[j]);
            result.emplace_back(sim, i, j);
        }
    }
    
    // 按相似度升序排序
    std::sort(result.begin(), result.end(),
              [](const auto& a, const auto& b) {
                  return std::get<0>(a) < std::get<0>(b);
              });
    
    return result;
}
```

**关键点：** Jaccard相似度 = $\frac{|A \cap B|}{|A \cup B|}$，用 `unordered_set` 实现交集计算 O(n)；用 `std::sort` 实现升序输出。

---

### 6. 写一个ROS/ROS2的rosbag数据读取与写入函数

**解答：**

```cpp
// === ROS1 版本 ===
#include <ros/ros.h>
#include <rosbag/bag.h>
#include <rosbag/view.h>
#include <sensor_msgs/PointCloud2.h>

void copyRosbag(const std::string& input_bag, const std::string& output_bag,
                const std::string& input_topic, const std::string& output_topic) {
    rosbag::Bag in_bag, out_bag;
    
    in_bag.open(input_bag, rosbag::bagmode::Read);
    out_bag.open(output_bag, rosbag::bagmode::Write);
    
    rosbag::View view(in_bag, rosbag::TopicQuery(input_topic));
    
    for (const rosbag::MessageInstance& msg : view) {
        sensor_msgs::PointCloud2::ConstPtr cloud = 
            msg.instantiate<sensor_msgs::PointCloud2>();
        if (cloud) {
            out_bag.write(output_topic, msg.getTime(), cloud);
        }
    }
    
    in_bag.close();
    out_bag.close();
}

// === ROS2 版本 ===
#include <rclcpp/rclcpp.hpp>
#include <rosbag2_cpp/reader.hpp>
#include <rosbag2_cpp/writer.hpp>

void copyRosbag2(const std::string& input_bag, const std::string& output_bag,
                 const std::string& input_topic, const std::string& output_topic) {
    rosbag2_cpp::Reader reader;
    reader.open(input_bag);
    
    rosbag2_cpp::Writer writer;
    writer.open(output_bag);
    
    while (reader.has_next()) {
        auto msg = reader.read_next();
        if (msg->topic_name == input_topic) {
            msg->topic_name = output_topic;
            writer.write(msg);
        }
    }
}
```

**参考：**
- [rosbag C++ API (ROS1)](http://wiki.ros.org/rosbag/Code%20API)
- [rosbag2 (ROS2)](https://github.com/ros2/rosbag2)

---

### 7. 计算数组容器的均值和协方差

**解答：**

```cpp
#include <vector>
#include <Eigen/Dense>

// 给定N个d维数据点，计算均值和协方差
std::pair<Eigen::VectorXd, Eigen::MatrixXd> 
computeMeanAndCov(const std::vector<Eigen::VectorXd>& points) {
    int N = points.size();
    int D = points[0].size();
    
    // 均值
    Eigen::VectorXd mean = Eigen::VectorXd::Zero(D);
    for (const auto& p : points) {
        mean += p;
    }
    mean /= N;
    
    // 协方差矩阵
    Eigen::MatrixXd cov = Eigen::MatrixXd::Zero(D, D);
    for (const auto& p : points) {
        Eigen::VectorXd diff = p - mean;
        cov += diff * diff.transpose();
    }
    cov /= (N - 1);  // 无偏估计：除以N-1
    
    return {mean, cov};
}

// 如果数据是Eigen::MatrixXd（每列一个数据点）
std::pair<Eigen::VectorXd, Eigen::MatrixXd>
computeMeanAndCovMatrix(const Eigen::MatrixXd& data) {
    // data: D x N 矩阵，每列是一个数据点
    int D = data.rows();
    int N = data.cols();
    
    Eigen::VectorXd mean = data.rowwise().mean();
    
    Eigen::MatrixXd centered = data.colwise() - mean;
    Eigen::MatrixXd cov = centered * centered.transpose() / (N - 1);
    
    return {mean, cov};
}
```

---

### 8. 计算支持高维数组容器的均值和协方差

**解答：**

```cpp
#include <vector>
#include <Eigen/Dense>
#include <stdexcept>

// 支持任意维度的均值协方差计算
// 使用模板兼容 EIGEN 的 fixed/dynamic size
template<int D>
std::pair<Eigen::Matrix<double, D, 1>, Eigen::Matrix<double, D, D>>
computeHighDimStats(const std::vector<Eigen::Matrix<double, D, 1>>& points) {
    if (points.empty()) {
        throw std::runtime_error("Empty point set");
    }
    
    int N = points.size();
    
    // 均值
    Eigen::Matrix<double, D, 1> mean = Eigen::Matrix<double, D, 1>::Zero();
    for (const auto& p : points) {
        mean += p;
    }
    mean /= static_cast<double>(N);
    
    // 协方差
    Eigen::Matrix<double, D, D> cov = Eigen::Matrix<double, D, D>::Zero();
    for (const auto& p : points) {
        Eigen::Matrix<double, D, 1> diff = p - mean;
        cov += diff * diff.transpose();
    }
    cov /= static_cast<double>(N - 1);
    
    return {mean, cov};
}

// 使用示例
std::vector<Eigen::Vector3d> points3d;
// ... 填充数据
auto [mean3d, cov3d] = computeHighDimStats<3>(points3d);
```

---

### 9&10. 空间平面拟合 + 点到平面距离

**解答：**

```cpp
#include <Eigen/Dense>
#include <Eigen/SVD>
#include <vector>

// 平面方程: n·p + d = 0，其中 n 是单位法向量
struct Plane {
    Eigen::Vector3d normal;  // 单位法向量
    double d;                 // 偏置
};

// 用SVD拟合空间平面
Plane fitPlaneSVD(const std::vector<Eigen::Vector3d>& points) {
    int N = points.size();
    
    // 计算质心
    Eigen::Vector3d centroid = Eigen::Vector3d::Zero();
    for (const auto& p : points) {
        centroid += p;
    }
    centroid /= N;
    
    // 构建去质心矩阵 A (3 x N)
    Eigen::MatrixXd A(3, N);
    for (int i = 0; i < N; i++) {
        A.col(i) = points[i] - centroid;
    }
    
    // SVD分解
    Eigen::JacobiSVD<Eigen::MatrixXd> svd(A, Eigen::ComputeFullU);
    
    // 最小奇异值对应的左奇异向量 = 法向量
    // (因为它对应于最小的方差方向)
    Eigen::Vector3d normal = svd.matrixU().col(2);
    normal.normalize();
    
    // 确保法向量方向一致（例如指向z轴正方向）
    if (normal.z() < 0) {
        normal = -normal;
    }
    
    double d = -normal.dot(centroid);
    
    return {normal, d};
}

// 点到平面距离（有符号）
double pointToPlaneDistance(const Eigen::Vector3d& point, const Plane& plane) {
    return std::abs(plane.normal.dot(point) + plane.d);
}

// 带符号距离（正=法向量方向一侧）
double signedPointToPlaneDistance(const Eigen::Vector3d& point, const Plane& plane) {
    return plane.normal.dot(point) + plane.d;
}
```

**原理：** SVD拟合平面的核心是找到数据**方差最小的方向**作为法向量。去质心后，协方差矩阵的最小特征值对应的特征向量就是法向量方向。

---

### 11. 写一个基于三维空间点的operator<函数

**解答：**

```cpp
#include <Eigen/Dense>

struct Point3D {
    double x, y, z;
    
    // 字典序比较：先x，再y，最后z
    bool operator<(const Point3D& other) const {
        if (x != other.x) return x < other.x;
        if (y != other.y) return y < other.y;
        return z < other.z;
    }
};

// 如果用Eigen::Vector3d作为key，不能直接修改Eigen，需要包装
struct EigenPointCompare {
    bool operator()(const Eigen::Vector3d& a, const Eigen::Vector3d& b) const {
        if (a.x() != b.x()) return a.x() < b.x();
        if (a.y() != b.y()) return a.y() < b.y();
        return a.z() < b.z();
    }
};

// 使用：std::map<Eigen::Vector3d, int, EigenPointCompare> pointMap;

// 也可以按其他规则排序（如按距离原点距离）
struct Point3DWithDist : Point3D {
    bool operator<(const Point3DWithDist& other) const {
        double d1 = x*x + y*y + z*z;  // 平方距离（避免sqrt）
        double d2 = other.x*other.x + other.y*other.y + other.z*other.z;
        return d1 < d2;
    }
};
```

**关键点：** `operator<` 必须满足**严格弱序**（Strict Weak Ordering）：非自反性、传递性、可比性。字典序是最常用的严格弱序实现。

---


## 十二、项目深挖、系统调试与行为面试题

### 1. 请用三分钟介绍你的SLAM项目。【高频】

建议结构：

1. 场景和需求：传感器、平台、实时性和精度目标；
2. 基线：采用什么前端、后端、地图；
3. 你的职责：明确到模块和代码；
4. 最大问题：漂移、动态、延迟或退化；
5. 解决方案：模型、实现和实验；
6. 结果：ATE/RPE、FPS、内存、成功率；
7. 反思：仍有哪些限制。

**模板：**“我负责XX系统的跟踪与局部建图。基线在动态场景中由于XX导致ATE为XX。我通过XX检测、XX加权和XX线程优化，将ATE降低XX%，前端耗时从XXms降到XXms。当前限制是XX场景下仍会退化。”

---

### 2. 你遇到过最难定位的Bug是什么？

优秀回答不是“代码写错一行”，而是完整排查链路：现象、假设、日志、最小复现、根因、修复、回归测试。

典型SLAM Bug：位姿方向反了、四元数顺序错误、纳秒/秒单位错、IMU时间偏移、Eigen未对齐、数据竞争、边缘化变量顺序错、深度尺度错。

---

### 3. 轨迹突然发散，你会如何排查？【高频】

1. 检查输入时间戳、频率、丢帧；
2. 检查标定、坐标系、单位；
3. 分别关闭视觉/IMU/回环定位问题；
4. 查看内点数、残差、bias、速度和协方差；
5. 检查初始化和退化运动；
6. 检查优化Hessian条件、NaN和阻尼；
7. 回放固定bag，定位首次异常帧；
8. 保存对应点和局部地图做离线复现。

---

### 4. 如何判断是前端问题还是后端问题？

- 用GT pose或固定后端测试前端匹配；
- 可视化对应点、重投影、光流和RANSAC内点；
- 将前端观测喂给离线优化器；
- 用合成无噪声数据验证后端Jacobian；
- 如果原始观测已错误，后端只能有限兜底；
- 如果观测正确但状态跳变，重点检查状态排序、线性化和边缘化。

---

### 5. 如何验证相机—IMU外参和时间偏移？

- 使用Kalibr等离线标定；
- 比较视觉角速度与IMU角速度；
- 做多轴旋转和8字运动；
- 检查重投影残差随图像位置和运动速度的系统偏差；
- 对时间偏移做网格搜索，看总残差是否有清晰最小值；
- 在线估计结果应在多段数据上稳定，而不是随轨迹任意漂移。

---

### 6. 如何调IMU噪声参数？

从数据手册或Allan方差得到noise density和bias random walk，按算法期望的连续/离散形式正确转换。再通过创新序列、NEES/NIS、bias曲线和不同运动数据验证。

不要靠“轨迹看起来更平滑”单独判断。过小的噪声会导致过度自信，过大则使观测更新过强、状态抖动。

---

### 7. 动态场景下如何保证系统鲁棒？

分层处理：

- 前端几何RANSAC；
- 语义或运动掩码；
- 残差鲁棒核；
- 动态概率时序更新；
- 地图只融合长期稳定区域；
- 保留足够静态背景特征；
- 失败时冻结建图并依赖IMU短时传播。

说明准确率和实时性的取舍，避免“把所有可动物体类别全部删除”。

---

### 8. 长走廊、隧道、纯平面场景退化怎么办？

- Hessian特征值检测退化方向；
- 在弱方向降低更新或增加先验；
- 融合IMU、轮速、GNSS；
- 增大局部地图或使用结构线/平面；
- 主动改变运动方向；
- 对回环候选进行更强几何验证，避免感知混淆。

---

### 9. 如何优化实时性能？【高频】

先profile，不要凭感觉优化。常见热点：图像金字塔、特征匹配、KD-tree搜索、残差线性化、稀疏求解、渲染和内存拷贝。

措施：减少拷贝、预分配、SoA布局、SIMD/TBB/OpenMP、局部地图、关键帧、并行化独立残差、合适求解器、异步回环、GPU kernel融合。

回答要给出优化前后毫秒和硬件。

---

### 10. 如何排查内存不断增长？

- 地图点/关键帧是否真正删除；
- shared_ptr循环引用；
- 队列是否无界；
- 可视化和日志是否缓存全部帧；
- GPU tensor/texture是否释放；
- Ceres Problem是否反复追加残差；
- 使用ASan、Valgrind、heaptrack或Massif。

---

### 11. 多线程地图访问如何保证安全？

明确读写所有权：跟踪读取地图快照，建图线程独占修改，回环通过版本或事务提交修正。避免大锁覆盖整次优化，可用读写锁、不可变快照、双缓冲和消息传递。

必须考虑对象生命周期，而不仅是容器加mutex。

---

### 12. 如何设计SLAM回归测试？

- 数学单测：SO(3)/SE(3)、Jacobian、预积分；
- 模块测试：特征跟踪、PnP、配准；
- 合成数据：已知GT和可控噪声；
- 固定bag回放：确定性参数和随机种子；
- 指标阈值：ATE、RPE、成功率、耗时、内存；
- 故障注入：丢帧、时间偏移、动态物体、传感器中断；
- 不同编译模式和硬件。

---

### 13. 如何做消融实验？

固定数据、随机种子、训练/评测协议，只改变一个模块。报告均值和方差，而不是单次最好值。除了最终ATE/PSNR，也要报告中间量，例如内点率、深度一致性、地图大小和运行时间。

若多个模块有交互，应增加组合实验，避免把协同效果错误归因于单一模块。

---

### 14. 指标提升很小，如何证明模块有价值？

分析模块针对的特定失败场景，而不是只看全数据平均。可报告：

- 极端场景成功率；
- P95误差；
- 跟踪丢失次数；
- 收敛速度；
- 计算开销；
- 对参数和噪声的敏感性。

如果提升小且开销大，应诚实说明不值得保留。

---

### 15. 线上系统和论文原型最大的区别是什么？

线上系统更重视：最坏延迟、内存上限、异常输入、自动恢复、日志可观测性、配置管理、版本兼容、长时间稳定性和传感器质量变化。

论文原型可能只需要在固定数据集跑通，线上系统必须处理所有“不应该发生但一定会发生”的情况。

---

### 16. 你如何进行参数管理？

参数分为标定参数、传感器噪声、算法阈值和性能预算。记录单位、默认值、有效范围、依赖关系和版本。关键参数应在启动时校验，并保存到结果目录以保证实验可复现。

---

### 17. 如何判断一个改进是否过拟合数据集？

- 跨数据集、跨场景和跨传感器测试；
- 不同随机种子；
- 不只调整测试序列参数；
- 检查参数敏感性；
- 在合成数据上验证理论行为；
- 测试与假设相反的场景。

---

### 18. 如果让你接手陌生SLAM代码库，第一周怎么做？

1. 跑通官方数据和指标；
2. 画线程、数据流和状态图；
3. 找入口、状态结构、残差和地图类；
4. 加时间统计和关键日志；
5. 建立最小可复现bag；
6. 阅读论文时对照具体函数；
7. 先修小Bug或补单测，再做结构修改。

---

### 19. 面试官质疑你的方案“为什么不用更简单的方法”怎么回答？

先承认基线价值，再说明你的场景约束和失败模式。比较简单方法与方案在精度、计算、维护和数据需求上的差异，并给出消融结果。不能只说“论文更先进”。

---

### 20. 你的项目还有哪些不足？【高频】

选择真实但可控的不足，例如：动态极端场景、长时间回环、显存、跨设备标定或极低纹理。说明你已定位根因、尝试过什么、下一步怎样验证。

避免回答“没有不足”，也不要暴露完全没有思路的致命问题。



## 十三、LeetCode算法题重点

### 高频重点题（加粗标记）

**数组与矩阵：**
- **[LeetCode 48. Rotate Image](https://leetcode.com/problems/rotate-image/)** — 旋转图像（90度），先转置再水平翻转
- **[LeetCode 73. Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes/)** — 矩阵置零，用第一行/列做标记
- **[LeetCode 54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix/)** — 螺旋矩阵遍历，四边界收缩

**搜索与回溯：**
- **[LeetCode 200. Number of Islands](https://leetcode.com/problems/number-of-islands/)** — ⭐ 岛屿数量，DFS/BFS/并查集（高频！）
- **[LeetCode 79. Word Search](https://leetcode.com/problems/word-search/)** — 单词搜索，DFS回溯
- **[LeetCode 46. Permutations](https://leetcode.com/problems/permutations/)** — 全排列，回溯模板题

**图论：**
- **[LeetCode 207. Course Schedule](https://leetcode.com/problems/course-schedule/)** — ⭐ 课程表，拓扑排序/环检测（因子图优化的基础！）
- **[LeetCode 743. Network Delay Time](https://leetcode.com/problems/network-delay-time/)** — Dijkstra最短路径

**递归/树：**
- **[LeetCode 104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/)** — 二叉树最大深度
- **[LeetCode 102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/)** — 层序遍历

**滑动窗口：**
- **[LeetCode 76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)** — 最小覆盖子串，滑动窗口模板
- **[LeetCode 15. 3Sum](https://leetcode.com/problems/3sum/)** — ⭐ 三数之和，排序+双指针（高频！）

**SLAM传感器相关（重点！）：**
- **[LeetCode 149. Max Points on a Line](https://leetcode.com/problems/max-points-on-a-line/)** — ⭐⭐⭐ 直线上最多的点（点云共线检测核心！极高频！）
- **[LeetCode 20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)** — 有效的括号（栈基础）
- **[LeetCode 49. Group Anagrams](https://leetcode.com/problems/group-anagrams/)** — 字母异位词分组（哈希）

**排序/查找：**
- **[LeetCode 33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/)** — 旋转数组查找，二分变体
- **[LeetCode 56. Merge Intervals](https://leetcode.com/problems/merge-intervals/)** — ⭐ 合并区间（点云聚类中常用！）
- **[LeetCode 215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)** — 快速选择 QuickSelect

---


## 十四、模拟面试快速追问题库

下面的问题适合口头快速复习。每题应能在30秒内先给结论，再根据面试官兴趣展开。

### 数学与优化

1. 为什么旋转矩阵有9个元素但只有3个自由度？
2. $SO(3)$ 的指数映射在小角度时如何近似？
3. 为什么Hessian秩亏？如何固定Gauge？
4. GN在什么情况下不如LM稳定？
5. 为什么鲁棒核需要作用在马氏距离上？
6. QR消元和Schur Complement各有什么数值特点？
7. 如何验证手写SE(3) Jacobian？
8. 边缘化后为什么不能随意改变旧线性化点？

### VIO与视觉SLAM

9. 单目纯旋转能否三角化？
10. 双目为什么天然有尺度？
11. IMU预积分为什么依赖bias线性化点？
12. 加速度计测到的是重力加速度还是比力？
13. VIO中全局yaw为什么不可观？
14. MSCKF为什么需要camera clone？
15. OpenVINS为什么强调FEJ？
16. VINS-Mono为什么使用4DoF pose graph？
17. ORB-SLAM3的Atlas解决了什么问题？
18. Basalt为什么叫square-root VIO？
19. KLT与ORB匹配如何组合使用？
20. 回环候选检索成功但几何验证失败的原因？
21. 相机曝光变化如何影响直接法？
22. Rolling shutter怎样进入投影模型？

### LiDAR与融合

23. Point-to-plane ICP在哪些方向可能退化？
24. FAST-LIO为什么适合高点数观测？
25. LIO-SAM和FAST-LIO后端最大的区别？
26. 点云去畸变为什么需要每个点的相对时间？
27. GNSS Float解是否应该完全丢弃？
28. 如何检测错误回环因子？
29. Scan Context为什么通常要搜索yaw对齐？
30. TSDF截断距离如何影响表面？

### C++与工程

31. Eigen对象放入STL容器时要注意什么？
32. shared_ptr循环引用怎么发生？
33. 为什么回调中不应持锁做优化？
34. vector扩容为什么会使指针失效？
35. 如何让多线程SLAM安全退出？
36. ROS2 QoS不匹配有什么表现？
37. 如何测量端到端延迟而不是单函数耗时？
38. 如何设计可复现的bag评测脚本？
39. ASan、TSan、UBSan分别检查什么？
40. release模式正常、debug模式正常但优化后崩溃，可能有哪些未定义行为？

### 项目追问

41. 你的方法在哪个序列失败最严重？为什么？
42. 你做的模块是否在所有场景都有效？
43. 你如何证明提升不是参数调出来的？
44. 如果性能预算减半，你先删哪个模块？
45. 如果没有GT，你如何评价系统是否变好？
46. 如果相机暂时失效2秒，系统怎么处理？
47. 如果IMU饱和，如何检测和恢复？
48. 如何避免错误位姿污染长期地图？
49. 你最想重构项目中的哪部分？
50. 如果重新做一次，你会改变哪个技术决策？



## 十五、参考资料汇总

- [SLAM十四讲 (高翔)](https://github.com/gaoxiang12/slambook2)
- [A micro Lie theory for state estimation in robotics (Sola, 2018)](https://arxiv.org/abs/1812.01537)
- [State Estimation for Robotics (Barfoot)](http://asrl.utias.utoronto.ca/~tdb/bib/barfoot_ser17.pdf)
- [Quaternion kinematics for the error-state Kalman filter (Sola, 2017)](https://arxiv.org/abs/1711.02508)
- [Probabilistic Robotics (Thrun, Burgard, Fox)](http://www.probabilistic-robotics.org/)
- [Ceres Solver 教程](http://ceres-solver.org/tutorial.html)
- [GTSAM 文档](https://gtsam.org/tutorials/)
- [PCL 教程](https://pcl.readthedocs.io/projects/tutorials/en/latest/)
- [Cartographer 文档](https://google-cartographer.readthedocs.io/)
- [C++ Reference](https://en.cppreference.com/)
- [原文知乎专栏 - SLAM相关技术学习与总结](https://www.zhihu.com/column/c_1527697103966461952)
- [原文知乎专栏 - 动态场景下SLAM的技术](https://www.zhihu.com/column/c_1527696022621319169)


### 扩展版新增推荐资料

- Timothy D. Barfoot, *State Estimation for Robotics*
- Joan Solà, *Quaternion Kinematics for the Error-State Kalman Filter*
- Joan Solà et al., *A micro Lie theory for state estimation in robotics*
- Forster et al., *On-Manifold Preintegration for Real-Time Visual-Inertial Odometry*
- Mourikis & Roumeliotis, *A Multi-State Constraint Kalman Filter for Vision-aided Inertial Navigation*
- VINS-Mono、ORB-SLAM3、OpenVINS、Basalt官方论文与代码
- Dellaert & Kaess, *Factor Graphs for Robot Perception*
- Ceres Solver、GTSAM、g2o、Sophus官方文档
- ROS2 Concepts、QoS、Executors、tf2与rosbag2官方文档
- 3D Gaussian Splatting原始论文及主流GS-SLAM开源实现

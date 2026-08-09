---
tags:
  - reviewer
---
## 致副主编的回复

我们感谢上一轮审稿中审稿人对我们工作的仔细和批判性评估。**根据审稿人的建议，我们对先前提交的稿件进行了实质性改进，包括引言、相关工作、方法、实验和结论部分，旨在更清晰地展示我们的方法并提供更全面的评估。**

在下文中，我们逐点回应审稿人的关切。审稿人的评论以**红色**高亮显示。

---

### 审稿人意见 P 1.1

> **所提出的系统依赖外部姿态估计模块（例如，VINS-Mono 和 ChArUco 标记）来提供准确的相机轨迹。虽然这一假设在静态建图场景中很常见，但在本文针对的半静态场景中存在问题，因为已知 SLAM 精度会显著下降。由于基于高斯溅射（Gaussian Splatting）的建图对姿态误差高度敏感，这一局限性直接影响了所提出框架的有效性和实际可部署性。**

**回复：**

我们感谢审稿人提出的这一重要意见。我们同意，姿态质量对任何基于 3DGS 的建图系统都是关键因素，严重的姿态误差会对地图质量产生负面影响。我们的工作并不旨在端到端解决完整的半静态 SLAM 问题；相反，它专注于在线半静态场景变化下的建图组件。

具体而言，VG-Mapping 采用双线程架构，包括一个跟踪线程和一个建图线程，其中跟踪线程使用 VINS-Mono，而我们提出的贡献在于建图模块。如论文所述，跟踪器在所提出的变分感知建图方法之外，建图线程以跟踪线程提供的相机姿态作为输入。

我们使用 VINS-Mono 的理由是，它基于局部时间测量进行短程视觉惯性状态估计，而不是基于先前构建的全局地图进行定位。因此，在我们的设定中，半静态变化主要挑战地图更新过程，这正是本文针对的问题，而它们对外部里程计模块的影响则不那么直接。尽管如此，我们并不声称该系统对严重的跟踪退化免疫，我们同意这一依赖性应在稿件中更明确地说明。

重要的是，我们论文中的真实世界基准已经使用跟踪线程估计的姿态（而非真实轨迹）来评估 VG-Mapping。此外，为了公平比较，我们记录了我们系统中估计的相机轨迹，并将其重用于所有基线方法，因此报告的性能提升可归因于所提出的建图方法，而非更有利的姿态来源。在真实基准上，我们的方法仍然在比较方法中实现了最佳的渲染质量，这为所提出框架的实用性和有效性提供了强有力的实证证据。

关于 ChArUco 标记，它们仅用于数据集采集 / 参考姿态配准，以减少构建基准测试时的混杂因素，而 VG-Mapping 的在线部署流程并不需要它们。运行时系统本身依赖外部跟踪器提供姿态输入。

在修订版中，我们在方法论和局限性讨论中都明确指出了这一点：VG-Mapping 是一个专为在外部提供的姿态下进行半静态场景更新而设计的建图框架，将其与更鲁棒的半静态 SLAM 或姿态精化后端集成是未来工作的一个重要方向。

---

### 审稿人意见 P 1.2

> **半静态能力的实验验证尚不充分。VG-Scene 数据集表现出的动态复杂性有限，且论文未提供几何精度或变化检测性能的定量评估，也未对不同类型的场景变化或失败案例进行分析。鉴于论文引入了一个新数据集并声称对场景变化具有鲁棒性，此类评估尤为重要。**

**回复：**

我们感谢审稿人提出的这一宝贵建议。我们同意，由于论文引入了一个新的半静态基准测试并声称对场景变化具有鲁棒性，因此需要更全面的实验验证。在修订版中，我们通过以下方式解决了这一问题：

首先，我们扩展了 VG-Scene 数据集，增加了更多样化的场景和半静态变化模式。实验结果还表明，这些案例对现有的在线 RGB-D 3DGS 基线方法仍然具有挑战性，后者在变化区域仍表现出陈旧结构或不完整的更新。

其次，为了解决审稿人对几何精度的关切，我们遵循常见做法，在渲染深度图上增加了两个基于深度的评估指标：Abs Rel（相对 $l_1$ 误差）和 $\delta_1$（阈值 1.25 下正确估计像素的百分比）。这些指标直接评估更新后的地图与变化后场景之间的几何一致性。新增的结果表明，由于更精确的地图更新，我们的方法实现了最低的渲染深度误差。

第三，关于变化检测性能，我们的方法并非设计为独立的对象级二值变化检测器。相反，VG-Mapping 通过提出的变化线索在像素级别执行在线变化感知地图更新。具体而言，外观变化模块的设计不仅响应实际场景变化，还突出显示重建不佳、需要进一步精化的区域。因此，传统的对象级变化检测指标（如精确率、召回率和 F1 分数）与我们的方法目标并不完全对齐。为了更好地解决这一关切，修订版稿件已通过基于深度的几何指标、不同场景变化类型的分析以及额外的定性讨论补充了原始评估。

最后，我们还加强了对不同变化模式的分析。具体而言，图 4 已经展示了两个代表性案例：一个是几何变化小但外观变化明显的案例，另一个是外观变化有限但几何变化清晰的案例。这些对应于所提出的基于外观和基于几何的变化线索的两个互补作用。修订版稿件现在明确阐述了这一关联，并更清晰地讨论了它们的效果。

---

### 审稿人意见 P 1.3

> **与现有半静态或连续 GS 基建图方法的比较仍然不足，难以清晰评估所提出方法相较于当前最新技术的优势。**

**回复：**

我们感谢审稿人指出这一点。我们同意，与近期半静态或连续 GS 基建图方法进行更清晰的比较，对于更好地评估所提出方法的优势是必要的。

在修订版稿件中，我们新增了一个最近开源的最先进半静态连续 GS 基建图方法——DynamicGSG，作为额外的基线。该比较在相同的实验设置和评估协议下进行，使得相对性能更具直接可比性。新结果表明，我们的方法始终优于这一近期方法，显示出对场景变化的更强鲁棒性和更精确的在线地图更新。因此，修订后的实验部分现在为 VG-Mapping 相较于当前最先进的半静态连续 GS 基建图方法的优势提供了更为清晰的图景。

---

## 摘要

维护能够准确反映环境近期变化的最新地图至关重要，特别是对于反复穿越同一空间的机器人而言。未能及时更新变化区域会导致地图质量下降，造成定位精度差、操作效率低下，甚至机器人丢失。三维高斯溅射（3DGS）最近因其密集、可微分和照片级真实的特性而在线地图重建中得到广泛采用，然而准确且高效地更新变化区域仍然是一个挑战。在本文中，我们提出了 VG-Mapping，一种专为此类半静态场景量身定制的新型在线 3DGS 基建图系统。我们的方法引入了一种**变化感知密度控制策略**，将高斯密度调节与优化解耦。具体而言，我们通过识别变化区域来引导初始化和剪枝，从而避免在定义后续优化的起点时使用陈旧信息。此外，为了解决该任务缺乏公开基准测试的问题，我们构建了一个包含合成和真实世界半静态环境的 RGB-D 数据集。实验结果表明，我们的方法在半静态场景中显著提升了渲染质量和地图更新效率。我们的代码和数据集已在 GitHub 上公开。

---

## 一、引言

同步定位与建图（SLAM）系统广泛应用于机器人、AR/VR 领域。传统 SLAM 算法 [1] 主要强调定位，建图仅作为支持定位的手段。这导致生成的稀疏地图不足以支撑除定位之外、需要稠密地图表示的下游应用。辐射场是最近提出的一种地图表示方法，为建图和渲染提供了可微分公式。它采用端到端优化来学习稠密场景表示，无需依赖稀缺的 3D 真实数据。在所提出的方法中，神经辐射场（NeRF）[2] 和三维高斯溅射（3DGS）[3] 最具代表性。与 NeRF 相比，3DGS 提供更快的优化速度、更高质量的渲染，以及更大的编辑灵活性，使其成为稠密建图的一个极具吸引力的解决方案。事实上，3DGS 最近已被集成到 SLAM 框架中，以克服稀疏地图的局限性 [4]，并作为下游机器人应用的地图表示。

在真实世界部署中，机器人频繁地反复访问同一地点。在重复访问期间，场景可能因人类活动（如家具重新摆放）而发生动态变化，形成所谓的**半静态场景**，如图 1（a）所示。从机器人的视角来看，半静态场景可被视为在不同时间点捕获的多个快照的组合。场景的先验地图由所有先前观测到的快照构建而成，而在已变化区域，先验地图与机器人当前传感器观测之间存在差异。请注意，半静态场景与动态场景在根本上有所不同，体现在两个方面：

1. 它们涉及**未观测到的过渡过程**，仅捕获物体位移的结果而非过程本身；
2. 它们表现出**与类别无关的变化**，因为可移动物体可以属于动物和车辆之外的多种类别。

由于这些特性，为动态环境设计的现有建图策略 [5] 不能直接应用于半静态场景。这些挑战凸显了需要能够在半静态条件下实时准确更新地图的建图系统，以支持下游机器人任务。

现在我们回到原始 3DGS。在该框架中，场景重建通过反向传播光度损失来优化高斯基元的属性，而自适应密度控制（ADC）则实现对基元的稠密化和剪枝。ADC 基于优化状态（如均值梯度和不透明度值）来精化高斯基元的数量。虽然这一策略对静态场景或离线全局优化有效，但它不太适合半静态场景中的在线建图。具体而言，由于其对场景变化的响应缓慢，ADC 无法及时调整变化区域中的高斯表示，导致新增内容缺失基元、移除结构产生错误基元。结果，优化在最坏情况下收敛到次优解，在最好情况下也需要大量额外迭代来补偿变化区域中的不一致性。

为解决这一局限性，近期工作探索了增强 3DGS 局部更新能力的方法。这些工作通常利用视觉基础模型 [6]、[7] 来比较同一视角下的渲染图像和观测图像，生成 2D 变化掩码。这些掩码随后被用于通过投票策略 [8]、基于掩码的损失函数 [9] 或对高斯基元进行对象级操作 [10] 来加速优化过程。尽管取得了进展，仍存在若干挑战：

- 首先，由于这些方法仍然依赖基于优化的 ADC，它们将变化区域中过时的高斯基元引入优化过程，这显著限制了变化区域的地图精度。
- 其次，在缺乏深度信息的情况下（这在机器人平台上通常可用），监督 3D 变化区域通常依赖批处理，从而限制了实时在线更新的可行性。

基于上述讨论，我们提出了 **VG-Mapping**，一种专为半静态环境量身定制的 RGB-D 在线 3DGS 基建图系统。在此类场景中，一个关键挑战在于维护一致且初始化良好的高斯表示，以应对场景变化。我们认为，有效的半静态在线建图关键在于**将密度初始化与优化过程解耦**，从而使新增或移除的结构能够以适当的初始条件进行处理。为实现这一解耦，VG-Mapping 引入了 **变化感知密度控制（VDC)** 机制，该机制引入了变化检测的关键步骤，以识别传入观测与现有地图之间的不一致性，从而引导有针对性的密度初始化。

虽然 3DGS 提供了照片级真实的外观建模，但其缺乏几何精度阻碍了深度线索在检测几何变化中的有效利用。此外，在这种非结构化、点状表示中进行变化检测的 3D 几何差分计算会带来显著的计算开销 [11]。为解决这些局限性，我们采用了一种**混合地图表示**，通过基于 TSDF 的体素地图来增强 3DGS 地图，为可靠的变化检测和高效的密度初始化提供细粒度的几何支持。

最后，为了解决半静态场景中在线 3DGS 建图缺乏公开 RGB-D 数据集的问题，我们构建了一个新的基准测试，包含六个模拟序列和三个真实世界序列。实验结果表明，VG-Mapping 能够有效利用深度信息，在半静态环境的变化区域中实现 3DGS 地图的高质量和高效率更新，如图 1（c）所示。

---

## 二、相关工作

### A. 地图表示

SLAM 作为机器人系统中不可或缺的感知模块，多年来得到了广泛研究。常见的地图表示包括基于稀疏特征的方法（如 ORB-SLAM [1]）、点云表示 [11]，以及稠密体积方法（如占用栅格 [12] 和截断符号距离函数（TSDF）[13]）。其中，TSDF 因其规则的网格结构、丰富的表面信息、对传感器噪声的鲁棒性、易于增量更新以及适合并行计算而受到广泛关注。然而，现有表示往往过于稀疏或缺乏物体搜索和路径规划等任务所需的照片级真实保真度。

最近，辐射场作为一种可微分且稠密的场景建图替代方案应运而生，其中 NeRF [2] 和 3DGS [3] 最具代表性。3DGS 使用 3D 高斯基元对场景进行建模，并采用可微分光栅化结合光度损失，通过反向传播优化这些基元，实现高保真新视角合成。由于其可微分性、稠密表示、照片级真实渲染和可编辑性，3DGS 作为地图表示迅速获得了关注。在本工作中，我们利用 TSDF 和 3DGS 的互补优势来构建一种混合表示，同时捕获精确的场景几何和照片级真实外观，并进一步利用 TSDF 提供的几何信息来引导 3DGS 中高效且准确的密度控制。

### B. 基于高斯的 RGB-D 在线稠密建图

由于其优势，3DGS 正越来越多地被集成到在线稠密建图系统中。Active3D [14] 引入了一种混合隐式-显式表示，将神经场与高斯基元融合，以联合捕获全局结构先验和局部观测细节。GSFusion [15] 结合了基于 TSDF 的表示来捕获表面几何，并采用四叉树图像分割策略来调节初始化高斯的数量。由于这些早期基于 3DGS 的 SLAM 算法假设场景是静态的，后续工作针对动态场景 [5]、[16]、[17]、[18]，旨在减轻移动物体对静态部分重建的影响。然而，这些方法仍然不适合半静态场景，后者以未观测到的过渡和与类别无关的变化为特征，这些变化通常不是由人或车辆引起的。在本工作中，我们专注于机器人领域中常见的半静态环境，并提出一种专为此类场景量身定制的基于高斯的 RGB-D 在线稠密建图系统。

### C. 3D 高斯的局部更新

对于半静态场景，最有效的地图更新策略是仅在已变化区域执行局部更新 [19]、[20]。CL-Splats [8] 和 GaussianUpdate [9] 利用视觉基础模型生成 2D 变化掩码，从而将联合优化限制在变化区域。然而，由于它们对深度信息的利用有限，这些方法通常依赖多视角观测或优化线索（如梯度或损失值）来识别需要更新的区域，这使得它们不适用于在线应用。DynamicGSG [10] 也使用语义级信息，首先将场景分割为物体，并为每个物体维护一组独立的高斯基元，以便后续更新可以在对象级别进行。由于 DynamicGSG [10] 对物体分割和跨帧匹配要求很高，它容易出现错误删除或不完整更新；此外，为了地图维护的目的，维护对象集比像素级局部更新更耗时。除了这些局限性，上述方法都依赖基于优化的 ADC，因此继承了前述的弱点。

---

## 三、方法论

与现代 SLAM 算法的典型架构一致，我们的系统由两个并行线程组成：**跟踪线程**和**建图线程**。跟踪线程采用 VINS-Mono [21] 进行实时相机姿态估计。由于 VINS-Mono [21] 以帧到帧的方式运行，其定位性能不受半静态环境中变化的影响。因此，我们只需专注于建图线程。

我们所提出的 VG-Mapping 流程如图 2 所示。给定时间步 $t-1$ 的 3DGS 地图 $\mathcal{G}_{t-1}$，以及时间 $t$ 的 RGB-D 观测 $\{\mathbf{I}_t, \mathbf{D}_t\}$ 和由跟踪线程提供的相应相机姿态 $\mathbf{T}_t$，我们的目标是高效更新地图中已变化的区域——特别是当前观测与 $\mathcal{G}_{t-1}$ 不一致的区域。这确保了机器人维护的地图保持最新，从而使环境变化不会对全局定位、避障和物体搜索等下游任务产生不利影响。

### A. 混合场景表示

为了增强 3DGS 地图捕获细粒度几何信息的能力，我们通过将基于 TSDF 的体素地图 $\mathcal{S}$ 与 3DGS 地图 $\mathcal{G}$ 相结合来构建混合表示。受 KinectFusion [13] 启发，我们整合从每帧深度图像导出的表面测量来构建全局 TSDF 体素地图。对于 3DGS 组件，我们通过反向传播优化高斯基元，使用由 RGB 和深度图像构建的损失函数。

#### 1）基于 TSDF 的体素地图

对于给定的体素大小 $s$，我们使用八叉树结构 [22] 构建表面体素地图，以加速体素访问。为了减少不必要的内存使用和计算开销，我们仅在表面附近分配体素。对于位于位置 $\mathbf{p} \in \mathbb{R}^3$ 的每个体素，存储两个分量：截断符号距离值 $F(\mathbf{p})$ 和权重 $W(\mathbf{p})$：

$$\mathbf{S}(\mathbf{p}) \mapsto [F(\mathbf{p}), W(\mathbf{p})]. \tag{1}$$

给定时间 $t$ 的深度图像 $\mathbf{D}_t$ 和相机姿态 $\mathbf{T}_t \mapsto [\mathbf{R}_t, \mathbf{t}_t]$，对于当前视锥体内每个 $\mathbf{p}$，体素 $\mathbf{S}_t(\mathbf{p}) \mapsto [F_t(\mathbf{p}), W_t(\mathbf{p})]$ 的值计算如下：

$$F_t(\mathbf{p}) = \phi\left(\mathbf{D}[\mathbf{u}] - \frac{\|\mathbf{t}_t - \mathbf{p}\|_2}{\|\mathbf{K}^{-1}(\mathbf{u}^\top|1)^\top\|_2}\right), \tag{2}$$

$$\mathbf{u} = \pi(\mathbf{K}\mathbf{T}_t^{-1}(\mathbf{p}^\top|1)^\top), \quad \phi(x) = \max(-1, \min(1, \frac{x}{\mu})), \tag{3}$$

$$W_t(\mathbf{p}) = \begin{cases} 1, & |F_t(\mathbf{p}) - F(\mathbf{p})| < \epsilon_F, \\ -5, & \text{otherwise}. \end{cases} \tag{4}$$

其中 $\mathbf{K}$ 是相机内参，$\mu$ 是截断值，$\pi(\cdot)$ 是包含反齐次化的透视投影。为了增强 TSDF 对半静态场景变化的响应能力，当从当前观测计算的 TSDF 值 $F_t(\mathbf{p})$ 与全局地图中的 TSDF 值 $F(\mathbf{p})$ 之间的差异超过阈值 $\epsilon_F$ 时，权重被设置为具有更大绝对值的负值。

全局 $\mathbf{S}(\mathbf{p})$ 通过 $\mathbf{S}_t(\mathbf{p})$ 更新如下：

$$F(\mathbf{p}) = \frac{|W_t(\mathbf{p})|F_t(\mathbf{p}) + W(\mathbf{p})F(\mathbf{p})}{|W_t(\mathbf{p})| + W(\mathbf{p})}, \tag{5}$$

$$W(\mathbf{p}) = \max(1, W(\mathbf{p}) + W_t(\mathbf{p})). \tag{6}$$

当 $|F_t(\mathbf{p}) - F(\mathbf{p})|$ 超过噪声范围 $\epsilon_F$ 时，我们推断该体素区域已发生变化。在这种情况下，$W(\mathbf{p})$ 减小，TSDF 值的可靠性降低。地图在每帧对视锥体内的所有体素进行更新，使其能够及时响应场景中的半静态变化调整 TSDF 值。

#### 2）3DGS

3DGS 使用 3D 高斯基元表示场景，每个基元由可优化变量参数化，包括均值 $\boldsymbol{\mu}$、分解为尺度矩阵 $\mathbf{S}$ 和旋转矩阵 $\mathbf{R}$ 的协方差矩阵 $\boldsymbol{\Sigma}$、不透明度 $\alpha$，以及用三阶球谐函数建模的视角相关颜色 $\mathbf{c}$。为了便于高效剪枝（如第 III-B 节所述），每个基元额外存储初始化时所属体素的 Morton 码 $V$。

给定帧 $t$ 的相机姿态 $\mathbf{T}_t \mapsto [\mathbf{R}_t, \mathbf{t}_t]$，每个 3D 高斯可通过溅射投影到图像平面，得到一个 2D 高斯 $\mathcal{N}(\boldsymbol{\mu}_{\mathbf{I}}, \boldsymbol{\Sigma}_{\mathbf{I}})$：

$$\boldsymbol{\mu}_{\mathbf{I}} = \pi(\mathbf{T}^{-1} \cdot \boldsymbol{\mu}), \quad \boldsymbol{\Sigma}_{\mathbf{I}} = \mathbf{J}\mathbf{R}_t^\top(\mathbf{R}\mathbf{S}\mathbf{S}^\top\mathbf{R}^\top)\mathbf{R}_t\mathbf{J}^\top, \tag{7}$$

其中 $\mathbf{J}$ 是将相机坐标转换为射线坐标的投影变换仿射近似的雅可比矩阵。基于相机坐标系中 3D 高斯的深度值 $z$，相应的 2D 高斯进行混合，像素颜色 $\bar{\mathbf{I}}[\mathbf{u}]$ 和深度 $\bar{\mathbf{D}}[\mathbf{u}]$ 渲染为：

$$\bar{\mathbf{I}}[\mathbf{u}] = \sum_{i}^{N} \mathbf{c}_i \alpha_i \mathcal{N}_i(\mathbf{u} \mid \boldsymbol{\mu}_{\mathbf{I}}, \boldsymbol{\Sigma}_{\mathbf{I}}) \prod_{j=1}^{i-1}(1 - \alpha_j \mathcal{N}_j(\mathbf{u} \mid \boldsymbol{\mu}_{\mathbf{I}}, \boldsymbol{\Sigma}_{\mathbf{I}})), \tag{8}$$

$$\bar{\mathbf{D}}[\mathbf{u}] = \sum_{i}^{N} z_i \alpha_i \mathcal{N}_i(\mathbf{u} \mid \boldsymbol{\mu}_{\mathbf{I}}, \boldsymbol{\Sigma}_{\mathbf{I}}) \prod_{j=1}^{i-1}(1 - \alpha_j \mathcal{N}_j(\mathbf{u} \mid \boldsymbol{\mu}_{\mathbf{I}}, \boldsymbol{\Sigma}_{\mathbf{I}})). \tag{9}$$

我们通过反向传播由渲染图像和捕获图像构建的损失来优化 3D 高斯基元：

$$\mathcal{L}_{rgb} = |\mathbf{I} - \bar{\mathbf{I}}|_1, \quad \mathcal{L}_d = |\mathbf{D} - \bar{\mathbf{D}}|_1. \tag{10}$$

与仅优化变化区域的方法不同，我们的方法利用所有图像信息进行优化。这有助于避免由变化区域引起的全局光照变化问题，以及因不准确的变化检测导致的遗漏区域问题。最新的 3DGS 地图使机器人能够在半静态环境中获取准确的照片级真实信息，这对于图像目标导航等高级任务至关重要。

### B. 变化感知密度控制

除了基于梯度的优化外，优化 3D 高斯基元的另一个关键步骤是自适应密度控制（ADC）。大多数基于原始 3DGS 的在线建图系统首先通过反向投影下采样深度图来初始化高斯基元，然后通过基于梯度的稠密化或剪枝来增加或减少其密度。然而，这种自适应机制在半静态场景中存在两个局限性：

1. 不区分变化区域和未变化区域，3DGS 的初始化导致未变化区域出现冗余基元、变化区域基元不足，这显著增加了后续 ADC 和优化过程的负担。
2. 需要迭代反向传播的基于梯度的稠密化和剪枝对实时显著地图变化的响应缓慢。

这些局限性导致半静态场景中性能次优且效率低下。为解决这些问题，我们提出了**变化感知密度控制（VDC）**，它：
1. 用改进的初始化方案替代稠密化；
2. 直接使用混合地图的几何信息剪枝高斯基元。

#### 1）初始化

受 GSFusion [15] 启发，我们首先基于像素值的均方误差使用四叉树对图像进行分割。四叉树的叶节点对应于方形图像块，在高纹理区域密集分布，而在低纹理区域（如墙壁），图像块相对稀疏且尺寸较大。然后，我们遍历每个叶节点执行基于外观和基于几何的变化检测。

**基于外观的变化检测（AVD）：** 使用 3DGS 先验地图在当前相机姿态下渲染图像，并计算与捕获图像的局部结构相似性指数（SSIM），以测量每个像素 $\mathbf{u}$ 处的外观差异：

$$\text{SSIM}(\mathbf{u}) = \frac{(2\mu_{\bar{\mathbf{I}}}\mu_{\mathbf{I}} + C_1)(2\Sigma_{\bar{\mathbf{I}},\mathbf{I}} + C_2)}{(\mu_{\bar{\mathbf{I}}}^2 + \mu_{\mathbf{I}}^2 + C_1)(\sigma_{\bar{\mathbf{I}}}^2 + \sigma_{\mathbf{I}}^2 + C_2)}, \tag{11}$$

$$\mu_{\mathbf{I}} = \frac{1}{|\mathcal{W}(\mathbf{u})|}\sum_{\mathbf{v} \in \mathcal{W}(\mathbf{u})} \mathbf{I}[\mathbf{v}], \tag{12}$$

$$\sigma_{\mathbf{I}} = \left(\frac{1}{|\mathcal{W}(\mathbf{u})| - 1}\sum_{\mathbf{v} \in \mathcal{W}(\mathbf{u})} (\mathbf{I}[\mathbf{v}] - \mu_{\mathbf{I}})^2\right)^{\frac{1}{2}}, \tag{13}$$

$$\Sigma_{\bar{\mathbf{I}},\mathbf{I}} = \frac{1}{|\mathcal{W}(\mathbf{u})| - 1}\sum_{\mathbf{v} \in \mathcal{W}(\mathbf{u})} (\bar{\mathbf{I}}[\mathbf{v}] - \mu_{\bar{\mathbf{I}}})(\mathbf{I}[\mathbf{v}] - \mu_{\mathbf{I}}), \tag{14}$$

其中 $\mathcal{W}(\mathbf{u})$ 表示以 $\mathbf{u}$ 为中心、大小为 $5 \times 5$ 的方形块。$C_1$ 和 $C_2$ 是避免零分母的常数。然后我们计算每个图像块的平均 SSIM 值。如果平均值低于阈值 $\tau_s$，则表明先前地图与当前传感器数据之间存在足够的不一致性，即半静态变化的迹象。值得注意的是，AVD 不仅检测外观已变化的区域，还识别当前 3DGS 地图难以重建的区域，从而在静态场景中也能提升渲染质量。

**基于几何的变化检测（GVD）：** 我们将每个图像块中心像素的深度值反向投影以获得相应的 3D 点。然后使用该 3D 坐标查询 TSDF 地图。如果该位置的体素权重大于 1，则该区域未发生变化，且该体素已用高斯基元初始化。在这种情况下，未发生半静态变化。GVD 防止在同一空间中进行冗余初始化，并聚焦于受场景变化影响的区域。

如果两种检测中的任意一种通过，我们为该图像块初始化一个高斯基元。均值使用通过反向投影图像块中心坐标 $\mathbf{u}_c$ 获得的表面位置 $\mathbf{p}_c$ 进行初始化。球谐函数的低频分量使用 $\mathbf{u}_c$ 处的 RGB 值初始化，其余分量设为零。不透明度设为 0.5，旋转矩阵设为单位矩阵。对于尺度矩阵 $\mathbf{S}$ 的初始化，我们同时考虑图像和几何信息：

$$\mathbf{S} = d \cdot \text{diag}\left(\frac{\mathbf{n}}{\|\mathbf{n}\|_2}\right), \quad d = \frac{L \cdot \mathbf{D}[\mathbf{u}_c]}{f_x}, \tag{15}$$

$$\mathbf{n} = \begin{cases} \mathbf{1}_3 \oslash (\mathbf{1}_3 + \text{abs}(\nabla \mathbf{S}(\mathbf{p}_c))), & \nabla \mathbf{S}(\mathbf{p}_c) \neq \emptyset, \\ \mathbf{1}_3, & \text{otherwise}. \end{cases} \tag{16}$$

其中 $f_x$ 指相机焦距，$L$ 表示图像块边长的一半，$\oslash$ 表示逐元素除法。$\nabla \mathbf{S}(\mathbf{p}_c)$ 表示通过对邻近体素的 TSDF 值取中心有限差分近似得到的 $\mathbf{p}_c$ 处的表面法线。$d$ 通过反向投影 $L$ 获得，反映图像纹理对尺度的影响。$\mathbf{n}$ 的引入使 3D 高斯的初始化偏向平贴于表面，从而减少实现精确对齐所需的优化步数。

#### 2）剪枝

利用混合地图的有序体素结构，我们在 TSDF 地图中高效执行光线投射操作，以识别分配在相机最小可测量范围 $n_p$ 和测量深度 $\mathbf{D}[\mathbf{u}]$ 偏移一个体素大小 $s$ 之间的体素：

$$\mathbf{r}_{\mathbf{u}} = \left\{ \mathbf{x}_t = z_t(\mathbf{K}^{-1}(\mathbf{u}^\top|1)) \;\middle|\; t \in \mathbb{Z}_{\geq 0}, \quad z_t = n_p + t \cdot s, \atop z_t < \mathbf{D}[\mathbf{u}] - s \right\}. \tag{17}$$

$\mathbf{r}_{\mathbf{u}}$ 中体素的 TSDF 值低于阈值 $\tau_p$ 表明该区域存在已删除的物体。通过计算这些体素的 Morton 码，并将其与高斯基元初始化时存储的 Morton 码进行匹配，我们可以高效识别相应的删除区域并对这些高斯进行剪枝。使用 Morton 码检索避免了逐点盒内检查的需要，提高了效率。

此外，考虑到定位误差和深度噪声可能导致初始化时体素和高斯基元被分配到不属于实际表面的区域，我们识别 TSDF 值大于 0.95 的体素，并剪枝其关联的高斯基元。得益于 TSDF 的噪声鲁棒性，这些体素的值快速收敛到 1。通过移除这些区域中相应的高斯基元，我们有效减少了由噪声引起的漂浮物。虽然深度噪声偶尔可能导致过度剪枝，造成受影响区域图像质量下降，但这一问题由 AVD 模块缓解。具体而言，AVD 检测这些外观退化并在相应区域重新初始化高斯基元，以恢复渲染质量。

与传统的 3DGS 剪枝（发生在优化之后）不同，我们的剪枝操作在当前帧优化之前进行。基于几何信息剪枝过时的高斯显著减少了优化步数，并消除了由过时信息引起的次优结果，从而确保对在线机器人任务（如定位）至关重要的准确且及时的地图更新。

#### 3）关键帧策略

对于每个输入帧，我们首先执行剪枝和初始化。如果当前帧中删除或添加的高斯数量超过阈值 $\tau_k$，则该帧被标记为关键帧，表明新观测到的信息或场景变化是实质性的。为确保关键帧沿轨迹均匀分布，如果当前帧与上一个关键帧至少相隔 10 帧，则该帧也被强制设为关键帧。当一帧被指定为关键帧时，它与从关键帧池中随机选取的关键帧子集进行联合优化；否则，仅当前帧参与 3DGS 的优化。

---

## 四、实验

### A. 实验设置

#### 1）数据集

现有用于评估半静态场景中 3DGS 的数据集 [8] 是为离线建图系统设计的。由于其视角稀疏且不连续、缺乏深度图像，它们不适合评估在线 RGB-D 3DGS 建图系统。为解决这些问题并实现对我们方法的全面评估，我们构建了一个开源数据集 **VG-Scene**，包含合成和真实世界基准测试。每个序列由同一场景的变化前子序列和变化后子序列组成。合成基准测试基于公开可用的 Blender 演示场景构建，我们定义相机轨迹并引入物体变化，共产生六个序列。真实世界基准测试包含三个序列，范围从桌面尺度到房间尺度，使用 Intel RealSense L515 捕获。VG-Scene 数据集涵盖了常见的场景变化类型，包括物体添加、移除和重新摆放。此外，我们在四个 ScanNet++ 序列（8b5caf3398、39f36da05b、b20a261fdf 和 f34d532901）上进行实验，遵循与 GS-Fusion 相同的协议，以进一步评估我们的方法在静态环境中的适用性。

#### 2）基线方法

我们将我们的方法与最先进的 RGB-D 在线建图系统进行比较，即 GS-SLAM [4]、GSFusion [15] 和 DynamicGSG [10]。为了公平比较，我们禁用了 GS-SLAM 和 DynamicGSG 中的跟踪模块，并使用来自 VINS-Mono [21] 的机器人位置信息。在评估期间，我们记录 VINS-Mono [21] 估计的相机轨迹，并在所有方法中重用它们。

#### 3）评价指标

遵循常见做法，我们使用 Abs Rel（相对 $l_1$ 误差）和 $\delta_1$（阈值 1.25 内正确估计像素的百分比）[23] 报告几何精度，并使用 PSNR、SSIM 和 LPIPS 评估渲染质量。此外，我们测量建图线程的运行时帧率以评估计算效率。表格中的最优和次优结果分别用粗体和下划线标出。

#### 4）实现细节

由于 GS-SLAM 的大量内存需求，所有实验均在单张 NVIDIA A6000 GPU 上进行。我们的系统使用 LibTorch 在 C++ 中实现，而 3DGS 光栅化和优化以及四叉树使用 CUDA 加速。对于所有序列，我们采用统一的参数配置：体素大小设为 0.01 m，基于外观的变化检测阈值为 $\tau_s = 0.6$，剪枝阈值为 $\tau_p = 0.2$，关键帧选择阈值为 $\tau_k = 200$。

### B. 地图更新性能

#### 1）半静态场景

我们在 VG-Scene 数据集上评估我们的方法，以评估其在半静态场景中的性能。所有方法遵循相同的实验协议，其中变化前子序列用于构建先验地图，变化后子序列用于更新它。定量结果总结于表 I，我们的方法在渲染质量上显著优于基线方法。与 DynamicGSG 相比，我们的方法将 PSNR 提高了约 50%，同时帧率提升了约 40 倍。此外，由于精确的地图更新，渲染深度表现出最低的误差。图 3 展示了定性比较，表明我们的解耦方法在变化区域产生高保真更新，即使在显著场景变化或具有噪声深度测量的真实世界场景中也是如此。实验表明，通过基于优化的方案更新变化区域使优化过程更难收敛，而对象级局部更新策略容易出现过度删除或不完整更新。

#### 2）静态场景

我们进一步在 ScanNet++ 数据集上评估我们方法的渲染质量，结果报告于表 II。尽管我们的框架主要为半静态环境设计，但它在静态设置中也带来了改进。这归因于基于外观的变化检测，它不仅识别动态变化，还捕获当前地图拟合不佳的区域。此外，剪枝阶段移除了远离表面的高斯，有效减少了漂浮伪影。这些机制共同作用，即使在完全静态场景中也提升了渲染质量。

### C. 消融研究

我们对初始化、剪枝模块进行消融研究，使用 VG-Scene 数据集中的三个序列（*Classroom*、*Cloister* 和 *Single table*）。结果总结于表 III。

#### 1）剪枝的效果

我们的剪枝策略在优化前高效移除过时的高斯，从而提升渲染质量。如图 4（a）所示，在观测有限的变化区域，其益处尤为显著。此外，通过直接存储 Morton 码并避免昂贵的点在盒内检查，我们的剪枝方法仅产生极小的计算开销。

#### 2）初始化的效果

表 III 的第二、第三和最后一行结果表明，AVD 和 GVD 均显著提升了地图更新的质量。AVD 便于检测不涉及几何位移的变化，而 GVD 解决外观相似但深度不同的区域的歧义，如图 4（b）和（c）所示。在变化区域进行有针对性的初始化在一定程度上增加了高斯基元的数量，进而导致系统帧率降低。

#### **3）评价指标** 

遵循常见做法，我们使用 Abs Rel（相对 l1​ 误差）和 δ1​ （阈值 1.25 内正确估计像素的百分比）[23] 报告几何精度，并使用 PSNR、SSIM 和 LPIPS 评估渲染质量。此外，我们测量建图线程的运行时帧率以评估计算效率。表格中的最优和次优结果分别用粗体和下划线标出。

#### **4）实现细节**

由于 GS-SLAM 的大量内存需求，所有实验均在单张 NVIDIA A6000 GPU 上进行。我们的系统使用 LibTorch 在 C++ 中实现，而 3DGS 光栅化和优化以及四叉树使用 CUDA 加速。对于所有序列，我们采用统一的参数配置：体素大小设为 0.01 m，基于外观的变化检测阈值为 τs​=0.6 ，剪枝阈值为 τp​=0.2 ，关键帧选择阈值为 τk​=200 。

### D. 应用

我们的研究动机在于提供一种能够处理半静态变化的准确且照片级真实的地图，以支持下游机器人任务。在本节中，我们在两个代表性的下游任务中评估我们所构建地图的有效性：开放词汇分割和 6D 姿态估计。

#### 1）开放词汇分割

由于 3DGS 能够从任意视角渲染照片级真实图像，多项研究利用 3DGS 作为 3D 地图与 2D 视觉基础模型（如 DINO [6]、SAM [7]）之间的桥梁，以支持图像目标导航和操控等任务。一个关键前提是渲染图像能作为视觉基础模型的有效输入。我们采用 Grounded-SAM [24] 对渲染图像进行开放词汇分割，并进行定性评估，如图 5 所示。得益于我们的方法在半静态环境中实现的精确场景变化更新，我们方法产生的渲染图像在分割物体方面远优于 GSFusion 的竞争方法。

#### 2）6D 姿态估计

近期研究 [25]、[26] 利用 3DGS 地图的可微分性和新视角渲染能力进行全局姿态估计。我们使用 GSFusion 和我们方法重建的 3DGS 地图作为 6DGS [25] 全局姿态估计的输入，并使用平均角误差（MAE）和平均平移误差（MTE）进行比较。实验在 VG-Scene 数据集上进行，使用两个合成序列和一个真实世界序列。对于每个合成序列，使用 Blender 在训练轨迹外渲染十张测试图像；对于真实序列，从变化后序列中均匀采样十张测试图像。如表 IV 所示，与基线相比，我们的方法实现了更低的姿态估计误差，展示了其在半静态场景中准确更新地图的能力，从而更好地支持下游机器人任务。

---

## 五、结论

我们提出了一种专为半静态环境量身定制的在线 RGB-D 3DGS 基建图系统。我们的方法通过结合 TSDF 和 3DGS 构建混合地图表示。通过利用 TSDF 地图的几何信息，我们引入了一个变化感知密度控制模块，使系统能够对经历半静态变化的区域以及重建不足的区域进行正确更新。此外，我们构建了一个半静态场景数据集来评估 RGB-D 3DGS 在线建图。实验结果表明，我们的系统在半静态环境中实现了高质量和高效率的地图更新，并提供了对下游机器人任务高度有益的最新 3DGS 地图。

在未来工作中，我们计划将所提出的建图系统扩展为完整的 SLAM 框架，并支持多种类型的深度传感器。此外，我们将把 VG-Scene 扩展到更大规模的室外环境，以便在更复杂和真实的场景中研究半静态变化。

---

## Summary

--- 

### Introduce

VG-Mapping 是一种专为半静态环境设计的在线 RGB-D 三维高斯溅射建图系统，旨在解决现有 3DGS 方法在半静态场景中因自适应密度控制机制（ADC）无法区分变化与未变化区域、且基于梯度的稠密化与剪枝响应迟缓而导致的地图更新不及时与优化收敛次优等问题。

系统首先构建了融合 TSDF 体素地图与 3DGS 地图的混合表示，以 TSDF 提供精确几何支撑与鲁棒增量更新能力，以 3DGS 保障照片级真实渲染质量，二者协同实现几何精度与视觉保真度的统一。系统接着通过提出变化感知密度控制机制，将密度初始化与优化过程解耦，引入基于外观的结构相似性指数检测（AVD）与基于几何的 TSDF 深度一致性检测（GVD）两种互补的变化识别模块，实现对变化区域的精准定位与针对性初始化，同时利用有序体素结构在优化前基于几何信息完成过时高斯基元的高效剪枝。

本文还构建了包含六个合成序列与三个真实世界序列的 VG-Scene 数据集，涵盖物体添加、移除与重新摆放等典型变化模式。

实验结果表明，VG-Mapping 在半静态场景中较现有最先进方法在渲染质量上提升约 50% 的同时实现约 40 倍的帧率提升，几何精度指标亦达到最优，且在静态场景与开放词汇分割、6D 姿态估计等下游任务中均展现出显著优势，验证了其在实际机器人应用中的有效性与实用价值。

### Contribution

- 提出一种变换感知密度控制机制，通过将密度初始化与优化过程解耦，并引入基于外观的结构相似性指数检测与基于几何的 TSDF 深度一致性检测两种互补的变化识别模块，实现了对半静态场景中变化区域的精准定位与针对性高斯基元初始化。
- 提出一种快速响应半静态场景变化的高效剪枝，利用 TSDF 几何信息在优化前完成过时基元的高效剪枝，从根本上解决了传统 3DGS 自适应密度控制在半静态环境中响应迟缓与初始化盲目的关键缺陷
- 针对在线 RGB-D 3DGS 半静态建图缺乏公开评估基准的问题，构建了包含六个合成序列与三个真实世界序列的 VG-Scene 数据集，涵盖物体添加、移除与重新摆放等典型半静态变化模式

### Advantage

- 变换感知密度控制机制创新性强
	将密度初始化和优化过程解耦，引入基于外观和几何的双重变化检测模块，实现了对变化区域的精准识别与针对性初始化

- 具有较好的泛化能力
	AVD和GVD模块可以识别重建不足区域，剪枝去除漂浮伪影，在静态数据序列上同样取得提升

- 计算效率与重建质量的提升
	优化剪枝策略避免了迭代反向传播的开销，Morton码检索机制替代逐点盒内检查，将剪枝计算开销降至极低

- 下游任务充分验证
	在开放词汇分割与 6D 姿态估计任务中均展现出显著优势，验证了实用价值

### Disadvantage

- 现有的实验对比在完整性上仍有提升空间。建议作者进一步引入若干代表性的静态建图方法，或增补在地图变更响应速度方面表现高效的先进基线算法作为对照，以进行更具说服力的定量评估。
	是否可以考虑对比 GPS-SLAM，该方法也是建图部分采用 TSDF 与 3DGS 混合地图，只不过是另一种混合方式，在效率上有明显提升。
	以及其他的静态场景下的 GS SLAM

- 虽然在审稿意见中提及VG-Mapping更关注于建图，跟踪部分使用 Vins-Mono，但是更高的位姿精度是否对于重建精度有提升
	是否可以将外观和深度的检测模块输出到前端跟踪，来识别出变化区域来提高跟踪精度
	或者更换适用半静态场景的跟踪方法来获取更高精度的跟踪

--- 

## Reviewer

---

### Organization and Style

- 结构严谨，逻辑连贯： 
	文章遵循标准的学术论文结构（摘要、引言、相关工作、方法论、实验、结论），从问题引入（传统3DGS在半静态场景的局限性）到解决方案（VDC和混合表示），再到实验验证，层层递进。

- 行文风格： 
	语言正式、精炼，学术表达准确，能够清晰地传达复杂的系统架构和算法逻辑。

### Technical Accuracy

- 数学公式严谨
	论文在描述TSDF体素更新、3DGS渲染以及SSIM计算时，采用了严谨的数学推导，公式变量定义清晰，技术细节完备。

- 机制设计合理 
	传统自适应密度控制（ADC）由于依赖梯度，在面临突发半静态变化时收敛极慢。本文引入TSDF不仅增强了对深度的利用和抗噪性，还能直接指导3DGS的初始化和剪枝，这一设计在理论和工程上均具有高度的合理性和可行性。
	AVD和GVD模块的设计具有互补性，AVD处理几何变化小但外观变化大的区域，GVD解决外观相似但深度不同的歧义。

### Presentation

- 实验设计全面： 
	评估不仅涵盖了地图更新的渲染质量（PSNR、SSIM、LPIPS）和几何精度（Abs Rel、$\delta_1$），还考虑了计算效率（FPS），对比维度丰富。
    
- 消融与下游任务验证： 
	消融实验有效证明了各个模块（初始化、剪枝）的必要性。此外，将生成的地图应用于开放词汇分割（Grounded-SAM）和6D姿态估计等机器人下游任务，极大地提升了论文的实用价值和应用前景。
	
- 基线对比较为充分：
	将VG-Mapping与GS-SLAM、GSFusion以及DynamicGSG进行了对比，在与DynamicGSG的对比中展现了显著优势，证明了方法的有效性。
	但是对比方法相对较少，

### Adequacy of Citations

- 文献覆盖全面： 
	相关工作部分梳理了地图表示（ORB-SLAM、KinectFusion、NeRF、3DGS）、基于高斯的RGB-D在线稠密建图（GS-SLAM、GSFusion等），以及3D高斯的局部更新（CL-Splats、GaussianUpdate、DynamicGSG）。
    
- 紧跟前沿： 
	引用了最新的视觉基础模型（如DINO、SAM）和最新开源的连续建图基线（DynamicGSG），证明作者对该领域的最新进展了如指掌，引用充分且具有针对性。

### Multimedia

- 图表应用合理：
	论文提到了多处关键图表支撑论点。例如图1展示了系统在半静态场景的优势；图2为清晰的系统流程图；图3和图4提供了直观的定性比较和消融可视化，帮助读者直观理解不同变化线索（AVD和GVD）的作用。
    
- 下游应用可视化： 
	图5展示了基于渲染图像的开放词汇分割结果，直观地证明了高保真地图对视觉模型的增益。

## Review Comments

1. 实验基线评测的完整性有待提升

论文虽然在半静态场景下进行了评测，但对比基线的选择范围仍显局限。建议作者在完全静态的基准序列下引入代表性的静态场景 GS-SLAM 方案（如 SplaTAM 或 MonoGS），以明确系统在追求动态适应性时是否牺牲了基础的视觉保真度。同时，应与同样采用 TSDF 与 3DGS 混合地图表征、但在融合机制与计算效率上表现优异的实时系统（如 GPS-SLAM）进行横向定量对比，以更具说服力地论证本方案在建图效率与场景更新敏捷度上的技术优势。

2. 跟踪与建图的单向解耦及位姿误差敏感性 

系统目前呈现“前端跟踪驱动后端建图”的严格单向数据流架构，其重建质量高度依赖外部里程计（如 VINS-Mono）提供的位姿输入。然而，传统里程计基于严苛的静态世界假设，在物体高频移动的半静态环境中极易产生轨迹漂移，而 3DGS 对位姿噪声极其敏感。系统未能将所提的外观（AVD）与几何（GVD）变化检测模块作为空间掩膜（Mask）反向反馈给前端跟踪以剔除动态特征，这种缺乏“建图-跟踪”双向闭环协同与联合优化的机制，限制了系统在复杂变更环境下的整体鲁棒性。

--- 

## Confidential Comments to the Editorial Staff

### Confidential Comments to the Editorial Staff (给副主编/编辑部的意见)

这篇文章主要解决的是 3DGS 在线建图时，遇到物体被挪动或拿走等“半静态场景”时更新不及时的问题。它的核心创新在于首次将体素与 3DGS 的混合地图模式应用到半静态场景中，并提出了一种变换感知密度控制机制。该机制通过解耦密度初始化与优化过程，融合外观与几何双重变化检测，实现了对变化区域的精准定位和高效高斯剪枝。在半静态场景中，该方法相比于目前的先进基线 DynamicGSG，在渲染质量提升约 50% 的同时，实现了约 40 倍的帧率提升。

总体而言，文章想法实用，创新点明确，性能提升显著。然而，该论文目前仍存在以下局限性：

第一，实验对比缺乏完整性。目前由于缺少与同类混合地图系统（如 GPS-SLAM）以及经典静态场景 GS-SLAM 方法的横向定量对比，无法充分验证其在基础视觉保真度上的表现。第二，系统采用严格的前端跟踪单向驱动后端建图的架构。在物体高频移动的半静态环境下，前端位姿极易产生漂移，而 3DGS 对位姿噪声高度敏感，但文中缺乏关于“更高精度的位姿”或“双向反馈闭环机制”是否能有效提升最终重建质量的深入分析与阐述。

鉴于这是一篇工程实现度很高的优秀论文，在上述问题得到作者的答复与澄清后，本研究适合被录用。

The manuscript primarily addresses the critical challenge of latent map convergence and delayed scene updates in online 3D Gaussian Splatting (3DGS) within semi-static environments undergoing object relocations or removals. The core innovation of this work lies in the pioneering application of a hybrid voxel-3DGS mapping paradigm to semi-static scenarios, supported by a novel change-aware density control mechanism. By decoupling density initialization from the continuous optimization pipeline and synergizing complementary appearance- and geometry-based change detection modules, the proposed framework achieves high-fidelity localization of altered regions and computationally efficient primitive pruning. Empirically, within semi-static regimes, the proposed method yields an approximate 50% improvement in rendering fidelity alongside a substantial 40x frame-rate acceleration over the state-of-the-art baseline, DynamicGSG.

Overall, the paper is conceptualized elegantly with salient innovations and compelling performance gains. Nevertheless, the manuscript currently exhibits the following technical limitations:

First, the comprehensiveness of the experimental validation requires enhancement. The absence of systematic benchmarking against analogous hybrid map architectures (e.g., GPS-SLAM) and classical static-scene GS-SLAM frameworks prevents a rigorous verification of its baseline visual fidelity. Second, the system architecture operates on a strict unidirectional data stream where front-end tracking drives back-end mapping. Given that front-end state estimation is highly susceptible to drift under frequent object displacements, and that 3DGS optimizations are inherently sensitive to pose noise, the manuscript lacks a rigorous analysis or explicit elaboration on whether higher pose accuracy or a bi-directional feedback loop could effectively mitigate this dependency to elevate the final reconstruction quality.

In summary, this work represents a well-executed and high-quality engineering effort. Therefore, the manuscript is recommended for publication, contingent upon the authors adequately addressing and clarifying the aforementioned technical concerns.

### Comments to the Author (给作者的意见)

本文提出了一种专门用于半静态环境的在线 RGB-D 3DGS 建图系统（VG-Mapping）。文章结构清晰，机制设计合理。**本工作最大的创新点在于，将 TSDF 体素与 3DGS 的混合地图模式巧妙应用于半静态场景，并提出了变换感知密度控制机制**。通过将初始化与优化过程解耦，结合外观（AVD）与几何（GVD）双重检测，实现了对变化区域的精准定位与基元更新；同时利用体素结构在优化前完成了过时基元的高效剪枝，有效解决了传统 3DGS 面对场景变化时响应迟缓的问题。此外，VG-Scene 数据集也为该领域提供了很好的评测基准。实验结果表明，该方法在半静态场景下相比于 DynamicGSG，在渲染质量和运行帧率上都取得了非常显著的提升。
为了进一步提升论文的质量，有以下几点建议供作者参考：

**第一，关于实验对比的完整性。** 虽然系统在与 DynamicGSG 的对比中展现了明显的优势，但整体基线范围仍有进一步拓展的空间。建议在完全静态的基准序列下，引入代表性的静态场景 GS-SLAM 方案（如 SplaTAM 或 MonoGS）进行对比，以明确系统在追求动态适应性时是否牺牲了基础的视觉保真度。同时，建议与同样采用混合地图表征且效率极高的实时系统（如 GPS-SLAM）进行横向定量对比，从而更具说服力地论证本方案在场景更新敏捷度上的技术优势。

**第二，关于位姿精度对后端重建质量的影响。** 目前系统采用“前端跟踪单向驱动后端建图”的架构，这意味着最终的重建质量高度依赖外部里程计（如 VINS-Mono）提供的位姿输入。在物体移动的半静态环境中，传统里程计容易产生轨迹漂移，而 3DGS 往往对位姿噪声非常敏感。因此，想请教并建议作者进行一些定性或定量的分析：更高精度的位姿是否能为系统最终的重建精度和渲染质量带来明显的提升？期望作者在最终版本中能对这一影响进行简单的讨论或分析说明。

This manuscript introduces an innovative online RGB-D 3DGS mapping framework (VG-Mapping) explicitly optimized for semi-static environments. Regarding Organization and Style, the paper is rigorously structured, and the technical exposition is formal, precise, and highly articulate. In terms of Major Contribution and Technical Accuracy, the core merit of this work resides in the elegant adaptation of a hybrid TSDF-voxel and 3DGS map representation to handle semi-static scene alterations via a change-aware density control mechanism. By decoupling the density initialization phase from the optimization pipeline and incorporating complementary appearance- (AVD) and geometry-based (GVD) detection strategies, the system effectively localizes changed spatial regions to guide targeted primitive instantiations. Concurrently, leveraging the ordered voxel topology to prune obsolete primitives prior to optimization effectively overcomes the slow-response bottleneck inherent in conventional 3DGS adaptive density control (ADC). Additionally, the newly curated VG-Scene dataset fills a notable void in standardized evaluation benchmarks for online RGB-D semi-static mapping. Regarding Presentation and Multimedia, the quantitative and qualitative results convincingly demonstrate that the proposed method substantially outperforms DynamicGSG in both rendering quality and throughput (achieving a 40x speedup) in semi-static settings.

To further enhance the academic rigor and completeness of the manuscript, the following constructive points are raised for the authors' consideration:

1. On the Comprehensiveness of Experimental Baseline Comparisons: While the system demonstrates a pronounced performance margin over DynamicGSG, the scope of the evaluation baselines should be expanded. It is highly recommended to evaluate representative static-scene GS-SLAM frameworks (e.g., SplaTAM or MonoGS) on completely static benchmark sequences to explicitly verify whether the system compromises baseline visual fidelity in pursuit of dynamic adaptability. Furthermore, a direct quantitative comparison should be conducted against real-time systems that also leverage hybrid map representations but feature optimized computational efficiency (such as GPS-SLAM), thereby providing a more compelling demonstration of the proposed method's technical advantages regarding scene update agility.
    
2. On the Impact of Pose Accuracy on Back-End Reconstruction Quality: The architecture currently enforces a strict unidirectional pipeline where front-end tracking drives back-end mapping, which implies that the final reconstruction fidelity remains upper-bounded by the pose trajectory supplied by an external odometer (e.g., VINS-Mono). In semi-static environments characterized by intermittent or frequent object displacements, traditional odometers are prone to tracking drift, an artifact to which 3DGS optimizations are notoriously sensitive. Consequently, the authors are encouraged to provide a deeper qualitative or quantitative analysis on the following critical aspect: Does higher pose accuracy yield a decisive or prominent improvement in the final reconstruction precision and rendering quality? A concise discussion or an analytical evaluation explicitly evaluating this sensitivity should be incorporated into the final version of the manuscript.
---
date: true
comment: true
totalwd: true
---

# MASt3R

!!! note "文章简介"
    Grounding Image Matching in 3D with MASt3R

    使用一个新的 head 去增强已有的 DUSt3R 网络获得 3D 特征，并提出快速匹配特征的算法进行 3D 视觉的图像匹配

## 简介

!!! note "一些概念"
    bag-of-keypoint 是一个特征向量，向量里每一位对应图像中的 descriptor 分配到一个 keypoint 的次数，可以用于图片分类

    https://www.cs.cmu.edu/~efros/courses/LBMV07/Papers/csurka-eccv-04.pdf

能够在同一场景的不同图像之间建立像素之间的对应关系（称为图像匹配）构成了所有 3D 视觉应用的核心组件，过去包括

- 依赖于离线映射阶段的图像匹配
    - [COLMAP](https://openaccess.thecvf.com/content_cvpr_2016/papers/Schonberger_Structure-From-Motion_Revisited_CVPR_2016_paper.pdf)
- 在线定位匹配
    - [PnP](https://dl.acm.org/doi/pdf/10.1145/358669.358692)

总体上分为三步流程<i id="keypoint-based-matching"></i>

1. 提取稀疏且可重复的关键点
2. 用局部不变特征描述它们
3. 通过比较特征空间中的距离来对离散关键点集进行配对

该方法优点在于

- 关键点检测器在低到中等照明和视点变化下是精确的
- 关键点的稀疏性使得问题在计算上易于处理

但是基于关键点的方法通过减少对 bag-of-keypoint 问题的关心，丢弃了对应任务的全局几何上下文，从而在重复 pattern 或低纹理区域的情况下特别容易出错。解决这个问题的方法包括

- 在配对步骤中引入全局优化策略，通常利用一些关于匹配的先验知识
    - 如果关键点及其描述符尚未编码足够的信息，那么在匹配期间利用全局上下文可能为时已晚
- 密集整体匹配，即完全避免关键点，一次匹配整个图像
    - 随着 global attention 的出现，已经有了一些成功的尝试，如 [LoFTR](https://arxiv.org/abs/2104.00680)

但是到目前为止，几乎所有匹配方法都将匹配视为图像空间中的 2D 问题；而实际上可以把匹配任务看作是一个 3D 问题，即对应的像素是相同 3D 点（ 2D 像素对应和 3D 空间中的相对相机姿态可以通过对极矩阵直接相关）。一个比较直接的证据是为 3D 重建构造的 DUSt3R 网络在 Map-free 定位基准测试中优于所有其他基于关键点和匹配的方法

因此，本文

- 在 DUSt3R 的基础上附加第二个 head 来回归密集的局部特征，输出局部特征图，可实现高度准确且极其稳健的 3D 感知与匹配
- 提出了一种与快速匹配算法及其从粗到细的匹配方案，能够处理高分辨率图像

## 相关工作

### 基于关键点的匹配（Keypoint-based matching）

一般分为 [上述的三步](#keypoint-based-matching)，且与以前的方法如 SIFT 不同，现代方法已经转向基于学习的数据驱动方案来检测或描述关键点，但正如上文所述，这些方法受到关键点的局部性的约束而丢弃了整体性，无法应对过大的视角转换

### 密集匹配（Dense matching）

这些方法旨在从全局角度考虑匹配，在详细的空间关系和纹理对于理解场景几何至关重要的场景中是非常有效的，但代价是需要增加计算资源。但这些方法仍然将匹配视为 2D 问题，限制了它们在视觉定位中的使用

### 相机姿态估计（Camera Pose estimation）

这方面技术差异很大，但在速度、准确性和鲁棒性权衡上最成功的策略基本上都是基于像素匹配的。匹配方法的不断改进促进了更具挑战性的基准测试，如定位数据集 Map-free，提供单个参考图像但没有地图，视点变化最高达 180°

### Grounding matching in 3D

传统 2D 方法受限的情况下， 3D 匹配是一个很有前景的方向。利用场景物理属性的先验来提高准确性或鲁棒性的方法已被广泛探索，但大多数先前的工作只是利用对极约束进行半监督对应学习，没有任何根本性的改变。而本文的贡献是通过回归局部特征并显式训练 DUSt3R 进行成对匹配，来实现 3D 匹配

## 方法

### 问题定义与概述

<figure markdown="span" id="figure-1">
![提议方法的概述](../../assets/img/docs/note/cv/MASt3R/mast3r_overview.svg)
<figcaption>图 1：提议方法的概述，与 DUSt3R 框架相比，本文的贡献以蓝色突出显示</figcaption>
</figure>

给定两张图像 $I^1$ 和 $I^2$，分别由两个未知参数的相机 $C^1$ 和 $C^2$ 捕获，希望找到一组对应关系 $\left\{(i, j)\right\}$ ，其中 $i$ 和 $j$ 分别是 $I^1$ 和 $I^2$ 中的像素坐标，$i = (u_i, v_i), j = (u_j, v_j) \in \left\{1, \ldots, W\right\} \times \left\{1, \ldots, H\right\}$，$W$ 和 $H$ 分别是图像的宽度和高度。为了简单起见，图片具有相同的分辨率，但不失一般性，最终的网络可以处理成对的可变纵横比图片

作者的方法如 [图 1](#figure-1) 所示，旨在联合执行 3D 场景重建并匹配给定的两个输入图像，基于 Wang 等人最近提出的 DUSt3R 框架

### DUSt3R 框架

!!! question "待了解内容"
    DUSt3R 怎么做 In the case where more than two images are provided, a second step of global alignment merges all pointmaps in the same coordinate system.

    Cross-attention 的实现方式

DUSt3R 仅从图像出发，解决联合标定和 3D 重建问题。大体过程是使用基于 Transformer 的网络预测两张图片的局部 3D 重建（local 3D reconstruction），并以稠密 3D 点云的形式 $X^{1,1}$ $X^{2,1}$ 输出，下记为点图（*pointmap*）

一个点图 $X^{a,b} \in \mathbb{R}^{H \times W \times 3}$ 代表着图片 $a$ 中每个像素 $i = (u, v)$ 与其对应的在相机 $C^b$ 坐标系中表示的 3D 点 $X_{u, v}^{a, b} \in \mathbb{R}^3$ 之间的稠密 2D-to-3D 映射关系

通过回归在相机 $C^1$ 坐标系中表示的 $X^{1,1}$ 和 $X^{2,1}$，DUSt3R 有效解决了联合标定和 3D 重建问题。在提供两个以上图像的情况下，全局对齐的第二步将合并同一坐标系中的所有点图，但**本文不考虑这种情况**

两张图片首先用 ViT 进行孪生（Siamese）方式的 encode，生成表示 $H^1$ 和 $H^2$
$$
H^a = \text{Encoder}(I^a) \quad \text{for} \quad a \in \{1, 2\}.
$$
然后使用两个交织在一起（intertwined）的 decoder 联合处理这些表示，通过 cross-attention 来理解视点和场景的全局 3D 几何形状之间的空间关系。用该空间信息增强的新表示表示为 $H'^1$ 和 $H'^2$
$$
H'^1, H'^2 = \text{Decoder}(H^1, H^2).
$$
最后，两个 prediction heads 根据 encoder 和 decoder 输出的级联表示回归最终点图和置信度图
$$
\begin{aligned}
X^{1,1}, C^1 &= \text{Head}^1_{3D}(\left[H^1, H'^1\right]), \\\\
X^{2,1}, C^2 &= \text{Head}^2_{3D}(\left[H^2, H'^2\right]).
\end{aligned}
$$

#### 回归损失

DUSt3R 在完全监督的情况下训练，采用如下的回归损失函数
$$
\ell_{regr} = \| \frac{1}{z} X_i^{v, 1} - \frac{1}{\hat{z}} \hat{X}_i^{v, 1} \|,
$$
其中

- $v \in \{1, 2\}$ 是视图
- $i$ 是一个像素，且该像素 ground-truth 3D 点 $\hat{X}^{v, 1} \in \mathbb{R}^3$ 已被定义
- 归一化因子 $z$ 和 $\hat{z}$ 为了使重建具有尺度不变性而被引入，被简单地定义为所有有效 3D 点到原点的平均距离

#### 度量预测

!!! question "不清楚的地方"
    Therefore, we modify the regression loss to ignore normalization for the predicted pointmaps when the ground-truth pointmaps are known to be metric.

这项工作注意到尺度不变性不一定是可取的，因为一些潜在的用例，如无地图视觉定位，需要度量可变的预测（metric-scale predictions）。因此当已知 ground-truth 的点图是度量时，我们修改回归损失以忽略预测点图的归一化，也即设置 $z := \hat{z} $ 并使得 $\ell_{regr} = \frac{\|X_i^{v, 1} - \hat{X}_i^{v, 1} \|}{\hat{z}}$。最终在 DUSt3R 中，包含置信度感知的损失函数为

$$
\mathcal{L}_{conf} = \sum_{v \in \{1, 2\}} \sum_{i \in \mathcal{V}^{v}} C_i^{v} \ell_{regr}(v, i) - \alpha \log C_i^{v}.
$$

### 匹配预测 head 和损失

为了从点图中获得可靠的像素对应，标准解决方案是在某些不变特征空间中寻找相互匹配。虽然这种方案即使在存在极端视点变化的情况下也能与 DUSt3R 的回归点图配合得非常好，但所得的对应关系相当不精确，从而产生了次优的精度。这是一个相当自然的结果，因为

- 回归本质上受到噪声的影响
- DUSt3R 从未经过明确的匹配训练

#### 匹配 head

基于这些原因，本工作添加第二个 head 来输出两个密集特征图 $D^1, D^2 \in \mathbb{R}^{H \times W \times d}$
$$
\begin{aligned}
D^1 &= \text{Head}^1_{desc}(\left[H^1, H'^1\right]), \\\\
D^2 &= \text{Head}^2_{desc}(\left[H^2, H'^2\right]).
\end{aligned}
$$
这个 head 的结构是使用非线性 GELU 激活函数的两层 MLP。最后，将每个局部特征标准化为单位范数

#### 匹配目标

作者希望鼓励一个图像中的每个局部描述符最多与另一幅图像中表示场景中相同 3D 点的单个描述符进行匹配

为了实现这个目标，使用 infoNCE 损失来处理 ground-truth 对应集 $\hat{\mathcal{M}} = \left\{(i, j)\right | \hat{X}_i^{1, 1} = \hat{X}_j^{2, 1}\}$

$$
\mathcal{L}_{match} = - \sum_{(i, j) \in \hat{\mathcal{M}}} \log \frac{s_\tau (i, j)}{\sum_{k \in \mathcal{P}^1 s_\tau (k, j)}} + \log \frac{s_\tau (j, i)}{\sum_{k \in \mathcal{P}^2 s_\tau (i, k)}},
$$

其中

- $s_\tau (i, j) = \exp [- \tau D_i^{1T} D_j^{2}]$
- $\mathcal{P}^1 = \left\{i | (i, j) \in \hat{\mathcal{M}}\right\}, \mathcal{P}^2 = \left\{j | (i, j) \in \hat{\mathcal{M}}\right\}$，代表每个图像中所考虑像素的子集
- $\tau$ 是温度超参数

这个匹配目标本质上是交叉熵分类损失，且与 $\ell_{regr}$ 不同，网络只有在正确的像素而不是附近的像素时才会获得奖励，鼓励了网络实现高精度匹配。最后结合回归和匹配的损失函数

$$
\mathcal{L}_{total} = \mathcal{L}_{conf} + \beta \mathcal{L}_{match}
$$

### 快速相互匹配

给定两个特征图 $D^1, D^2 \in \mathbb{R}^{H \times W \times d}$，目标是找到一组可靠的像素对应关系（彼此的相互最近邻居）
$$
\mathcal{M} = \left\\{(i, j) | j = \text{NN}_2(D_i^1) \text{ and } i = \text{NN}_2(D_j^2)\right\\},
$$
其中 $\text{NN}_A(D_j^B) = \argmin_{i} \left\|D_i^A - D_j^B\right\|$ 是 $D_j^B$ 在 $D^A$ 中的最近邻

但是相互匹配的简单实现具有很高的计算复杂度 $\Omicron(W^2 H^2)$，且一些优化（如 K-d 树）在高维特征空间中很低效

#### 快速匹配

因此，作者提出基于子采样（sub-sampling）的更快方法

这是一个基于从初始拥有 $k$ 个像素的稀疏像素集 $U^0 = \{ U_n^0 \}_{n=1}^k$ 开始的迭代过程，这个集合通常是在第一张图像的网格 $I^1$ 上规律性采样得到的。接着，每个像素被映射到它在 $I^2$ 上对应的最近邻像素，记作 $V^1$，然后被映射到的像素再以相同的方式重新映射回 $I^1$

$$
U^t \mapsto [\text{NN}_2(D_u^1)]_{u \in U^t} \equiv V^t \mapsto [\text{NN}_1(D_v^2)]_{v \in V^t} \equiv U^{t+1}
$$

然后收集一组相互匹配（形成循环的匹配，即 $\mathcal{M}_k^t = \left\{(U_n^t, V_n^t) | U_n^t = U_n^{t+1} \right\}$）。接着对于下一次迭代，已经收敛的像素被过滤掉，即更新 $U^{t+1} := U^{t+1} \setminus U^{t}$。类似的，从 $t = 1$ 开始也对 $V^{t+1}$ 进行验证和过滤，并以类似的方式将其与 $V^t$ 进行比较。如 [图 2](#figure-2) 所示，这个过程会迭代一个固定的数目，然后大多数对应情况会收敛到稳定的互相匹配状态。如 [图 3](#figure-3) 所示，可以看到未收敛点的数目 $|U^t|$ 在几次迭代之后迅速收敛到零。最后，输出的对应集是所有互相匹配对的并集 $\mathcal{M}_k = \cup_t \mathcal{M}_k^t$

<figure markdown="span" id="figure-2">
![快速匹配的图示](../../assets/img/docs/note/cv/MASt3R/reciprocal.svg)
<figcaption>图 2：快速匹配迭代过程</figcaption>
</figure>

<figure markdown="span" id="figure-3">
![快速匹配迭代过程中状态](../../assets/img/docs/note/cv/MASt3R/reciprocal_stat.svg)
<figcaption>图 3：快速匹配迭代过程中状态</figcaption>
</figure>

<figure markdown="span" id="figure-4">
![在 map-free 数据集上性能和时间对比](../../assets/img/docs/note/cv/MASt3R/time_versus_perf_mapfree.svg)
<figcaption>图 4：在 map-free 数据集上性能和时间对比</figcaption>
</figure>

#### 理论保证

快速匹配的总体复杂度为 $\Omicron(kWH)$，比朴素方法（被 all 标记）要快 $\frac{WH}{k} \gg 1$，如 [图 4](#figure-4) 所示

此外，该算法的结果是全集 $\mathcal{M}$ 的子集，大小 $|\mathcal{M}|$ 被限制在 $|\mathcal{M}| < k$

!!! note "关于证明"
    作者在补充材料中研究了该算法的收敛保证以及它如何表现出离群值过滤特性，解释了为什么最终精度实际上高于使用完整对应集 $\mathcal{M}$ 时的原因，此处不再详述

### 由粗到精的匹配

!!! question "待了解内容"
    Test-time resolutions

由于 attention 的二次复杂度，MASt3R 最大只能处理 512 个像素的区域，更大的图像需要更多算力来训练，且 ViT 还不能推广到更大的 test-time resolutions，而缩小比例进行匹配再放大回原先的分辨率会导致性能损失/定位精度或重建质量大幅下降

粗到精匹配是一种标准技术，可以获得对高分辨率图像的处理能力并保留低分辨率算法的优势

就 MASt3R 而言，先对降采样之后的图片进行一个粗略的匹配，其中子采样参数为 $k$，将其记为 $M_k^0$。接着，独立地在每个全分辨率图像上生成重叠的网格窗口 $W^1, W^2 \in \mathbb{R}^{w \times 4}$，每个窗口的最大尺寸为 512 像素且连续的窗口重叠部分大于 50%。然后就可以枚举所有的窗口对 $(w_1, w_2) \in W^1 \times W^2$，从中选出覆盖大多数粗略匹配 $\mathcal{M}_k^0$ 中的子集（具体过程也就是从窗口对中采用贪婪方式逐个进行选择，直到覆盖 90% 的粗略匹配）。最终，对各个窗口对进行精细匹配，并将从每个窗口对获得的对应关系最终被映射回原始图像坐标并连接，从而提供密集的全分辨率匹配
$$
\begin{aligned}
D^{w_1}, D^{w_2} &= \text{MASt3R}(I_{w_1}^1, I_{w_2}^2), \\\\
\mathcal{M}_k^{w_1, w_2} &= \text{fast\\_reciprocal\\_NN}(D^{w_1}, D^{w_2})
\end{aligned}
$$

## 实验

TODO

## 总结

TODO

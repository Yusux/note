---
date: true
comment: true
totalwd: true
---

# MCTSr

!!! note "文章简介"
    Accessing GPT-4 level Mathematical Olympiad Solutions via Monte Carlo Tree Self-refine with LLaMa-3 8B: A Technical Report

    本文介绍了 MCT Self-Refine (MCTSr) 算法，旨在结合蒙特卡洛树搜索提高复杂数学推理任务的性能

## 简介

GPT-4 和 LLaMA 等大语言模型以其数十亿参数架构为特征，表现出卓越的语言理解和生成能力，也提供了 NLP 之外如数学问题解决、推荐系统等任务的潜力。但当前一个重大障碍是输出的准确性和可信度。尤其是在精度至关重要的数学环境中，LLM 总是容易产生幻觉（hallucinations），即输出表面上看似合理，但实际上不正确，对理性过程有害的结果。尽管 Self-Refine 这样的重写技术可以缓解，但其性能仍然有限

因此本文提出了 MCTSr，旨在结合 LLM 和 蒙特卡洛树搜索，增强在复杂数学推理任务中的表现，主要贡献如下

- 将 LLM 与 UCT-MCTS 集成来开发并验证一种新颖的推理算法
- 提出了一个动态剪枝模块，可以细化 MCTS 框架内的决策过程，促进更高效、更准确的问题解决能力
- 通过广泛的实验，深入了解 LLM 和 MCTS 的协同潜力

## 方法

### 基础知识

#### MCTS

蒙特卡洛树搜索（MCTS）涉及四个关键阶段

- Selection：从根开始，算法根据特定策略（例如UCT）选择有希望的子节点，一直持续到到达叶节点
- Expansion：在叶节点处，除非它代表游戏的最终状态，否则将添加一个或多个可行的新子节点来说明未来可能的移动
- Simulation or Evaluation：从新添加的节点开始，算法通过任意选择动作进行随机模拟，直到得出游戏结论，从而评估节点的潜力
- Backpropagation：模拟后，结果（获胜、失败或平局）被反向传播回根，更新每个遍历节点的统计数据（例如，获胜、失败）以告知未来的决策

MCTS 通过反复迭代这些阶段，逐步构建决策树，在由于状态空间巨大而无法直接计算最佳策略的情况下细化最优决策的策略

#### UCT

应用于树算法的置信上限（UCT）通过选择最大化 $\text{UCT}_j = \bar{X}_j + C \sqrt{\frac{2 \ln N_c}{N_j}}$ 的动作来平衡探索和利用，其中

- $\bar{X}_j$ 是动作 $j$ 的平均奖励
- $N_c$ 是仿真总次数，等于所有 $N_j$ 的和
- $N_j$ 是动作 $j$ 的访问次数
- $C$ 是探索参数

#### MCT Self-Refine

MCT Self-Refine 算法代表了 MCTS 与 LLM 的集成，将数学问题解决方案的迭代细化过程抽象为搜索树结构，该树上的节点代表答案的不同版本，而边表示改进的尝试

为方便理解，提出以下符号：

- $P$：正在解决的问题实例
- $A$：节点集，每个节点代表 $P$ 的一个潜在答案
- $M$：每个节点可用的操作集，表示对答案可能的 Self-Refine 修改
- $R$：根据修改的质量和有效性对节点进行自我奖励采样的函数
- $R_a$：存储节点 $a$ 所有具有自我奖励函数 $R$ 的自我奖励采样结果的集合
- $T$：根据达到最大迭代次数或达到满意的答案质量等标准决定搜索过程终止的函数
- $Q(a)$：估计答案节点 $a$ 价值的价值函数，源自累积奖励 $R_a$ 和来自子节点的反向传播
- $U⁢(a)$：节点 $a$ 的 $Q$ 值的置信上限，以平衡利用和探索
- $\text{Father⁢}(a)$：返回给定节点 $a$ 的父节点的函数
    - 如果 $a$ 是根节点，则该函数返回 null 或特定标识符
- $\text{Children⁢}(a)$：返回给定节点 $a$ 的所有子节点集合的函数
    - 表示通过执行操作 $m \in M$ 后派生的所有可能状态
- $N⁢(a)$：节点 $a$ 的访问总数，用于计算其 UCB 值并评估利用和探索状态
    - 由于每次访问时都进行奖励采样，因此该值等于 $|R_a|$

### 算法

MCTSr 的主要工作流程结构如下

- Initialization：使用朴素的模型生成的答案和虚拟响应（例如“我不知道”）建立根节点，以最大限度地减少模型过度拟合的趋势
- Selection：算法采用价值函数 $Q$ 对所有未完全展开的答案进行排序，并使用贪心策略选择最高价值的节点进行进一步探索和细化
- Self-Refine：所选答案 $a$ 使用 Self-Refine 框架进行优化
    - 最初，模型生成反馈 $m$，指导 refine 过程产生增强的答案 $a'$
- Self-Evaluation：对 refine 的答案进行评分，以采样奖励值并计算其 $Q$ 值
    - 这涉及到模型的自我奖励反馈和严格的评分标准、抑制满分等约束，以保证评分的可靠性和公平性
- Backpropagation：将 refine 答案的值反向传播到其父节点和其他相关节点，以更新树的值信息
    - 如果任何子节点的 $Q$ 值发生更改，则父节点的 $Q$ 也会更新
- UCT Update：所有节点的 $Q$ 值更新后，我们确定候选节点的集合 $C$ 用于进一步扩展或选择，然后使用 U⁢C⁢T 更新公式以更新下一个选择阶段的所有节点的 U⁢C⁢T 值

该算法迭代这些阶段，直到满足终止条件 T ，包括推出约束或最大探索深度，不断改进答案的质量，并探索新的可能性

#### Selection

利用数学问题 refine 过程的马尔可夫性质，可以专注于**选择所有叶节点和那些未完全扩展的节点**，而忽略 refine 的历史路径。但考虑到在此任务中充当策略的 LLM 可以为任何答案状态 $a$ 生成无限数量的细化操作 $m$，每个节点都可能面临一组无限的扩展操作，因此借鉴贝叶斯优化中期望改进的概念，提出了确定“**完全扩展**”的两个标准

- 节点的子节点数量达到预定义的限制
- 至少有一个子节点的 Q 值超过了该节点的值

可以根据这些标准确定候选节点的集合 $C$ 以进行进一步扩展或选择

#### Self-Refine

在 Self-Refine 过程中，模型以多轮对话优化提示为指导，优化问题 $P$ 的答案 $a$。模型先生成关于 $a$ 的反思性或批评性评论 $m$，然后在 $m$ 的指导下，模型修改 $a$ 以生成改进版本 $a'$。这种迭代细化提高了响应的质量，利用结构化反馈来推动答案的演变

#### Self-Evaluation

根据随机过程的马尔可夫性质（只与当前状态有关），在数学问题 $P$ 的 refine 过程中，答案 $a$ 的 $Q$ 值被定义为将 $a$ 进一步细化为更优答案的预期质量。即与传统的 MCTS $Q⁢(s,a)$ 估计状态 $s$ 下动作 $a$ 的值不同，这里的 $Q⁢(a)$ 源自对 $a$ 的奖励函数值的多次采样。具体而言，该模型采用自我奖励的方法来估计 $a$ 的奖励，要求提供 -100 到 100 的奖励分数，且为了避免模型的奖励趋势过于平滑导致实践中答案之间缺乏比较区别，采样时使用如下三个约束

- Prompt Constraint：模型在奖励评分时必须遵守最严格的标准
- Full Score Suppression：指示模型不要提供满分的反馈分数
    - 任何高于 95 分的奖励评分都会按一定量减少，以抑制分数过高
- Repeated Sampling：每次访问搜索树节点都会对该节点的奖励进行重复采样，以增强自评估的可靠性
    - 当对某个节点的子节点进行奖励采样时，也会对其父节点进行奖励采样，以增加奖励采样的样本量

采样后，计算 $a$ 的 $Q$ 值，并为了抵消自我奖励函数的平滑趋势，在预期奖励中添加了最小值约束，平衡最坏情况和平均结果
$$
Q(a) = \frac{1}{2}(\min R_a + \frac{1}{|R_a|} \sum_{i=1}^{|R_a|} R_a^i)
$$

#### Backpropagation

在所有叶子节点的奖励值采样和 $Q$ 值更新完成后，会将这一变化传播到其父节点和祖先节点，即如果节点 $a$ 的子节点集合 $\text{Children⁢}(a)$ 中任意元素的 $Q$ 值发生变化，则该节点的 $Q$ 函数值更新为
$$
Q'(a) = \frac{1}{2}(Q(a) + \max_{i \in \text{Children}(a)} Q(i))
$$
其中 $Q'⁢(a)$ 是考虑其子节点影响的答案 $a$ 的更新质量值

#### UCT Update

使用结合 UCB1 方法的 UCT 来平衡节点的探索和利用；对于候选集合 $C$中的节点 $a$ ，其 $U⁢C⁢T_a$ 值为
$$
UCT_a = Q(a) + c \sqrt{\frac{\ln N(\text{Father}(a)) + 1}{N(a) + \epsilon}}
$$
其中 $\epsilon$ 是避免除零错误的小常数

根据候选集 $C$ 的 UCT 值，通过贪婪采样或重要性采样的方式来选择最优节点来继续探索 refine 过程

#### Termination Function

!!! question "不理解的地方"
    Advanced Criteria Based on Language Model Logits: The search concludes based on predefined metrics derived from the language model’s logits.

!!! note "rollout"
    The standard use of "rollout" (also called a "playout") is in regard to an execution of a policy from the current state when there is some uncertainty about the next state or outcome - it is one simulation from your current state. The purpose is for an agent to evaluate many possible next actions in order to find an action that will maximize value (long-term expected reward).

    [Tesauro and Galperin NIPS 1997](http://papers.nips.cc/paper/1302-on-line-policy-improvement-using-monte-carlo-search.pdf)

    > In backgammon parlance, the expected value of a position is known as the "equity" of the position, and estimating the equity by Monte-Carlo sampling is known as performing a "rollout." This involves playing the position out to completion many times with different random dice sequences, using a fixed policy P to make move decisions for both sides.


在 MCTSr 算法中，搜索终止函数标准 $T$ 包含如下几个条件

- Early Stopping：当搜索结果的不再改进或连续搜索产生重复结果时，搜索就会终止
- Search Constraints：一旦 rollout 的数量达到预定限制或树中的一个或多个节点满足最大深度约束，搜索就会终止
- Advanced Criteria Based on Language Model Logits：基于从语言模型逻辑推出的预定义指标，让搜索过程进行总结

一旦满足终止函数条件 $T$，就可以根据 $Q$ 值或其他条件从树节点收集最佳答案

## 实验

TODO

## 不足和未来工作

- 作为通用决策框架，MCTSr 在各种场景中的潜在应用还有待进一步探索
    - 例如黑盒优化问题和大型语言模型的自驱动对齐
    - MCTSr 的组件具有高度可扩展性，可以持续开发来识别和比较更广泛的组件算法，从而增强 MCTSr 算法的实际潜力和有效性

~~论文写作顺序看着有点难受，特别是算法那边最开始没讲 selection~~
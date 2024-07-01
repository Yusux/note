---
date: true
comment: true
totalwd: true
---

# RoboCasa

!!! abstract "简介"
    RoboCasa: Large-Scale Simulation of Everyday Tasks for Generalist Robots

    AI 的最新进展很大程度上是由规模化推动的，但是在 Robotics 中，由于不存在现实生活中大量的标注数据，难以进行规模化。因此本文提出以厨房环境为中心的逼真且多样化的场景以生成的机器人数据进行大规模模仿学习。

## Introduction

Robotics 的一个关键问题是如何获取体现现实世界的巨大多样性和复杂性的机器人训练数据，如

- RT-1: Robotics transformer for real-world control at scale
- Bridge data: Boosting generalization of robotic skills with cross-domain datasets
- Open X-Embodiment: Robotic learning datasets and RT-X models
- Droid: A large-scale in-the-wild robot manipulation dataset

虽然这些数据集可以在狭窄领域提高机器人的泛化能力，但是就通用而言，机器人学习规​​模化的可行路径是什么？这是本文研究的动机

而从方案而言，由于在现实世界中收集越来越大的数据集需要不切实际的大量资本和劳动力，因此可以使用模拟作为生成大量用于模型训练的合成数据的替代方案

- 一旦创建了功能丰富的高保真模拟器，我们就可以以低成本生成大量机器人数据
- 生成式人工智能的快速发展促进了真实模拟的创建
- 这可以加速 robot learning 的进程，让新 idea 可以快速实现原型并可复用

而要实现高质量模拟，模拟器必须满足三个核心标准

- 必须保证物理、渲染和底层模型的真实性，以便能够转移到现实世界
- 必须满足其提供的场景、资产和任务的多样性
- 必须伴随大型机器人数据集，以捕获其所提供的场景和行为的多样性

综上，本文提出以家庭环境为中心的、在 Robosuite: A modular simulation framework and benchmark for robot learning 之上、基于 MuJoCo 的大型模拟框架 RoboCasa，以训练 generalist robots。支持单臂移动平台、人形机器人和带手臂的四足机器人等实体。其中包含 100 项系统评估任务（前 25 项是具有基本机器人技能的原子任务，例如拾取和放置、打开和关闭门以及扭转旋钮，另外 75 项是涉及一系列机器人技能的复合任务），使用从 LLMs 获取活动列表的方法来生成任务。另外，也通过扩展 MimicGen 来为原子任务生成 100K 个额外的 trajectories

## Related Work

### 流行仿真框架

|Feature|RoboCasa|AI2-THOR|Habitat 2.0|iGibson 2.0|RLBench|Behavior-1K|robomimic|ManiSkill 2|OPTIMUS|LIBERO|MimicGen|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|Mobile Manipulation|✓|✓|✓|✓|✗|✓|✗|✓|✗|✗|✓|
|Room-Scale Scenes|✓|✓|✓|✓|✗|✓|✗|✗|✗|✗|✗|
|Realistic Object Physics|✓|✗|✗|✓|✓|✓|✓|✓|✓|✓|✓|
|AI-generated Tasks|✓|✗|✗|✗|✗|✗|✗|✗|✗|✗|✗|
|AI-generated Assets|✓|✗|✗|✗|✗|✗|✗|✗|✗|✗|✗|
|Photorealism|✓|✓|✓|✗|✗|✓|✗|✓|✓|✗|✗|
|Cross-Embodiment|✓|✓|✗|✓|✗|✓|✗|✗|✓|✗|✓|
|Num Tasks|100|-|3|6|100|1000|8|20|10|130|12|
|Num Scenes|120|-|1|15|1|50|3|-|4|20|1|
|Num Object Categories|153|-|46|-|28|1265|-|-|-|x|-|
|Num Objects|2509|3578|169|1217|28|5215|15|2144|72|x|40|
|Human Data|✓|✗|✗|✓|✗|✗|✓|✗|✗|✓|✓|
|Machine-Generated Data|✓|✗|✗|✗|✓|✗|✓|✓|✓|✗|✓|
|Num Trajectories|100K+|-|-|-|-|0|6K|30K|245K|5K|50K|

### Robotics 数据集

论文发表前，有几项机器人方面大规模数据收集工作

- 自监督学习，通过试错来收集抓取和推动等任务的数据
    - 由于试错过程，可能需要花费大量时间才能生成高质量数据
- 通过人类远程操作，控制机器人引导其完成不同的任务来收集数据
- 在模拟中利用 algorithmic trajectory generators
    - 通常利用特权信息和手工设计的启发式方法，需要大量人工参与

### 从大型离线数据集中学习

- Behavioral Cloning
    - 训练策略来模仿数据集中的操作
- Offline Reinforcement Learning
    - 尝试使用奖励函数来优先选择某些数据集操作而不是其他操作

##  RoboCasa 模拟

### 核心模拟平台

因为 RoboSuite 注重物理真实性、高速和模块化设计，方便扩展到大型场景。此外，为了支持房间规模的环境，作者扩展了 RoboSuite 以适应各类形态

另外，也使用 NVIDIA Omniverse 进行高质量渲染以捕捉逼真的图像

### 厨房场景

在 RoboCasa 当前版本中，专注于以厨房活动为中心的家务任务，包含各类布局设计、可交互器具与纹理（MidJourney 生成）

![厨房布局设计](../../assets/img/docs/PR/Robotics/RoboCasa/image.png)

![可交互设备](../../assets/img/docs/PR/Robotics/RoboCasa/image-1.png)

### 资产库

作者使用从 Objaverse 数据集和 Luma.ai（文本转 3D）获取到包括橱柜、抽屉和各种厨房用具（水果和蔬菜、乳制品、家禽、饮料、容器、工具等）的资产，进行存储并在使用时转换为 MuJoCo MJCF 模型格式

![资产](../../assets/img/docs/PR/Robotics/RoboCasa/image-2.png)

## RoboCasa 活动数据集

TODO
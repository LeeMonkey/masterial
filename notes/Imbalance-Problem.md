# Imbalance Problems in Object Detection: A Review

## 0. Instroduction

1. Class Imbalance  
类别不平衡，主要有样本数量的差异引起

2. Scale Imbalance  
尺度不平衡，主要有目标的尺度引起

3. Spatial Imbalance  
空间不平衡，不同样本对回归（Regression）损失贡献的不平衡，IoU上的不平衡，物体分布位置的不平衡

4. Objective Imbalance  
多任务损失优化之间的不平衡，最常见于分类误差和回归误差之间的差异

## 1. Objective Imbalance  

分类损失比回归损失大，回归只是基于前景样本做回归，例如Faster R-CNN算法RoI-subnetwork阶段，分类损失是回归的2~4倍。
  
最常见的方法是设置weighting factor进行调整，观察模型的最优情况，另一种解决方案是[Prime Sample Attention in Object Detection](https://arxiv.org/abs/1904.04821)里面的**Classification-Aware Regression Loss**，想让网络更关注回归较好的Bounding-Boxes，因此让回归损失和分类得分相关，从而使梯度从Regression Branch流到Classification Branch  

作者在open issue章节中也提到，这种关联应该被更加深入的探索。典型的例子就是[Towards Accurate One-Stage Object Detection with AP-Loss](https://arxiv.org/abs/1904.06373)

## 2. Spatial Imbalance  

大小，形状，位置（相对于图像或者其他box）以及IoU都是Bounding Boxes的空间属性，这些属性不平衡可能会影响训练或者泛化的性能。例如，位置上的小偏差可能导致回归（定位）误差的剧烈变化，从而导致误差值的不平衡。  

### 2.1 Imbalance in Regression Loss  

训练过程中，不同的正例对Regression Loss的贡献是不同的，对于低质量IoU的bboxes会产生很大的损失，L1和L2 Loss会被这些bboxes所主导。  
Smooth L1 Loss由Fast R-CNN提出，降权了那些过大损失的样本（可能是离群点outliers）。  
Balanced L1 Loss由[Libra R-CNN](http://openaccess.thecvf.com/content_CVPR_2019/papers/Pang_Libra_R-CNN_Towards_Balanced_Learning_for_Object_Detection_CVPR_2019_paper.pdf)提出，提升了inliers的权重，进一步降低了outliers的权重。其他Loss如下所示：

Loss Function         | Explanation  
--                    | --  
*L*2 Loss             | 应用在早期的深度目标检测器，应用在小误差上</br>很稳定但是更偏重对outliers的惩罚
*L*1 Loss             | 对小误差不稳定
Smooth *L*1 Loss      | 回归损失函数的基准，相较于L1 Loss对outliers更鲁棒。
Balanced *L*1 Loss    | 相较于Smooth L1 Loss增加了inliers的贡献
Kullback-Leibler Loss | 预测基于KL散度的输入BBOX的置信度
IoU Loss              | 使用IoU间接计算作为loss函数
Bounded IoU Loss      | 除了在反向传播过程中梯度需要估计的参数以外，固定用IoU定义的输入框的所有参数
GIoU Loss             | 根据输入到IoU的最小外接矩阵扩展IoU的定义，</br>然后直接使用IoU和GIou（扩展后的IoU）作为损失函数

### 2.2 IoU Distribution Imbalance  

IoU不平衡是指Bounding Boxes在IoU段的分布上呈现出明显的不均匀的分布，Libra R-CNN和Cascade R-CNN都探讨过这个问题。在负例上，IoU在0~0.1范围内的样本占据主导，在正例上，IoU在0.5~0.6之间的样本占据主导。  

作者推荐的工作是[Cascade R-CNN](https://arxiv.org/abs/1712.00726)（[Naiyan Wang: CVPR18 Detection文章选介（上）](https://zhuanlan.zhihu.com/p/35882192)），通过级联结构，逐步提高IoU threshold，增强正样本的质量，防止Regressor对单一阈值过拟合。

### 2.3 Object Location Imbalance

基于anchor的检测器将anchor boxes均匀地排布在图像上，但是物体不一定服从这种分布，如下图所示：

<center><img alt="Distribution of the centers of the objects in the MSCOCO dataset over the normalized image" src="https://pic4.zhimg.com/80/v2-91dee22ca92cf959379a0b7a035b4177_720w.jpg"/><br/>Distribution of the centers of the objects in the MSCOCO dataset over the normalized image</center>  

这里作者推荐[Region Proposal by Guided Anchoring](https://arxiv.org/abs/1901.03278)的工作（[Kai Chen:Guided Anchoring：物体检测器也能自己学习Achor](https://zhuanlan.zhihu.com/p/55854246)）。作为一个Anchor-free的RPN，它可以预测出proposals的位置，如下图中的箭头所示：

<center><img alt="Guide Anchoring" src="https://pic2.zhimg.com/v2-6127807a220533da786c737ef42c3d5d_r.jpg"/></center>

## 3. Scale Imbalance

### 3.1 Object/Box-level Scale Imbalance

当某个尺度范围内的物体over-represent该数据集后，Scale Imbalance就会发生。[An Analysis of Scale Invariance in Object Detection - SNIP](https://arxiv.org/abs/1711.08189)中的Investigation指出这种Imbalance会极大影响overall detection performance。下图表达了COCO数据集中长宽面积上的不平衡：

<center><img alt="Inbalance of Width, Height and Area" src="https://pic1.zhimg.com/v2-1bc0bbee141ab43d5bacd8010d8242bc_r.jpg"/></center>

为了处理多样性的边界框，Pyramid方法是最常用的。包括image pyramid（SNIP, SNIPER），feature pyramid（SSD，FPN等），以及feature pyramid + image pyramid，作者将[TridentNet](https://arxiv.org/abs/1901.01892)（[Naiyan Wang：TridentNet：处理目标检测中尺度变化新思路](https://zhuanlan.zhihu.com/p/54334986)）列为这方面的典型工作。

<center><img alt="Image Pyramid and Feature Pyramid" src="https://pic4.zhimg.com/v2-6ce3ec843795fdba8d7b32f09d618a17_r.jpg"/><br/>(a) No method. (b) Backbone pyramids.(e.g., SSD) (c) Feature pyramids (e.g., FPN).(d) Image pyramids (e.g., SNIP). (e) Image and feature pyramids. (e.g. TridentNet)</center>

### 3.2 Feature-level Imbalance

这种不平衡主要指FPN-based architecture里，层级之间特征的不平衡，Low Level和High Level的特征之间户有定位/语义之间的优缺点，如何mitigate这种不平衡来达到最佳的检测效果，而解决方案也大多是结构上的，来看看下面各式各样的连接方法：

<center><img alt="FPN-based architecture" src="https://pic3.zhimg.com/80/v2-e99dd871c228fb88472503cad1c297d2_720w.jpg"/></center>

## 4. Class Imbalance

### 4.1 Foreground-Foreground Class Imbalance

这一类型的不平衡指的是分类类别不平衡，在数据集或者是一个batch都会存在。但是这一类型的不平衡并没有太大的引起现阶段目标检测研究的重视。

### 4.2 Foreground-Background Class Imbalance

这是目标检测中研究最广泛，程度最深的一类不平衡。这种平衡并不是由于数据集引起，而是由现有目标检测架构引起：background anchors远远多于forground anchors。似乎自deep detectors诞生以来，人们就移植在努力去克服这种不平衡。

<center><img alt="Foreground-Background Class Imbalance" src="https://pic4.zhimg.com/v2-361aa868149fa2c07fbf561936b30f4b_r.jpg"/></center>

作者将这类不平衡的方法分为两类：

(1) hard sampling：可以理解成有偏采样。包括由mini-batch undersampling（R-CNN系列标准配置），OHEM，IoU-balanced sampling， PISA等，作者在这里将类RPN的objectness方法也归结为了这一类。

(2) soft sampling：可以理解为loss reweighting。最著名的方法莫过于Focal Loss。

<center><img alt="soft sampling" src="https://pic4.zhimg.com/80/v2-ef9c070e304aff6a40e7ec3b88a6031f_720w.jpg"/></center>

遗憾的是，由于时间的缘故，这篇综述中并没有对最新的 anchor-free 检测器进行分析。但是个人认为 anchor-free 的 detector 存在着类似的不平衡。例如，anchor-free 的检测器大多基于关键点的检测驱动，如 extreme point，center point，corner point；其中，foreground points 数量比 background points 存在着明显差异，虽然可能不若 anchor boxes 那般造成如此剧烈的不平衡，但是这仍然导致绝大部分的 anchor-free 检测器采用了 Focal Loss 或者其变体来训练网络。
我们可以看到，几乎所有的 sampling heuristics 都基于启发式，并且具有大量的超参数需要调整。例如，OHEM 需要调 mini-batch size 和 fraction，Focal Loss 需要调 α 与 γ，GHM 有一些必要的前提假设与区间 M 需要调。正如 GHM 文中所说，解决不平衡最佳的策略（分布）是难以定义的。因此，我们重新回顾了 foreground-background imbalance，来探讨 sampling heuristics 是否必要。我们认为，不像一般的，由数据引起的不平衡，foreground-background imbalance 在训练和测试中是具有等同分布的；而使用采样，可能会改变这种分布，并不一定会在测试中取得更好的结果；但不使用采样，就会陷入难以训练的境地。我们发现，从初始化，损失，推理三个方面辅以适当的策略，即可在没有任何 sampling heuristics 的情况下，总是可以达到更好的检测精度。我们开源了相关代码（[https://github.com/ChenJoya/sampling-free](https://github.com/ChenJoya/sampling-free)），希望多多讨论，互相启发。

## 5. Conclusions

目标检测中的不平衡问题是一个古老的问题，自检测器诞生之初，人们就在与其战斗。Imbalance Problems in Object Detection: A Review 的作者总结了不平衡的各种类型，并且详细分析了已经出现的研究，还在 open issue 中给出了悬而未解的问题。十分推荐。

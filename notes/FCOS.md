# FCOS-Fully Convolution One-Stage Object Detection（单阶段全卷积目标检测）

## 摘要

FCOS实现了anchor box free、proposal free、避免了关于anchor的复杂计算，减少训练过程中的内存消耗，后处理阶段只采用NMS，比一般的基于anchor的目标检测算法要更简单

## 基于anchor检测算法的缺点

1. 需要谨慎调整与anchor的尺寸、长宽比及数量相关的超参数
2. 对形变较大的目标泛化不好
3. 增加anchor数量容易造成正负样本不平衡
4. anchor增加计算量

## FCOS优势

1. FCOS与FCN思想一致，可以复用这些算法的设计理念
2. 实现了proposal free和anchor free，显著减少设计参数的数量。避免复杂的IoU计算以及训练期间anchor box与groundtruth box的匹配
3. FCOS可以作为两阶段任务中的RPN网络
4. FCOS可拓展至其他任务（实例分割，关键点检测）


## FCOS算法详解

<center><img alt="the Architecture of FCOS" src="https://pic4.zhimg.com/80/v2-bbec7cf563fc6c0bb981bef30bb9ca17_720w.jpg"/></center>

FCOS首先使用Backone CNN(用于提取特征的主干架构CNN)，另s为feature map之前的总步长。

### 1) 单阶段全卷积检测器

1. feature map映射

    feature map中位置为$(x,y)$，映射到输入图像的位置为$(\lfloor \frac{s}{2} \rfloor + xs, \lfloor \frac{s}{2} \rfloor + ys)$

2. 正负样本定义

    在FCOS中，如果位置$(x, y)$落入任何真实边框，就认为它是一个正样本，它的类别标记为这个真实边框的类别。
    > 这样会带来一个问题，如果标注的真实边框重叠，位置 [公式] 映射到原图中落到多个真实边框，这个位置被认为是模糊样本，后面会讲到用多级预测的方式解决的方式解决模糊样本的问题。

3. 分类器

    FCOS训练$C$个二元分类器(C是类别的数目)

4. 损失函数

    FCOS算法的损失函数：
    $$L \left(\left\{ p_{x,y} \right\}, \left\{t_{x,y}\right\}\right) = \frac{1}{N_{pos}} \sum_{x,y}L_{cls}\left(p_{x,y}, c_{x,y}^{*}\right) + \frac{\lambda}{N_{pos}}\sum_{x,y}1_{\{c_{x,y}\}>0}L_{reg}(t_{x,y}, t_{x,y}^{*})$$

### 2) FPN多级预测

首先明确两个问题：

1. 基于锚框的检测器由于大的步幅导致低召回率，需要通过降低正的锚框所需的交并比分数来进行补偿：在FCOS算法中表明，即使是大的步幅(stride)，也可以获取较好的召回率，甚至效果可以优于基于锚框的检测器。

2. 真实边框中的重叠可能会在训练过程中造成难以处理的歧义，这种模糊性导致基于fcn的检测器性能下降：在FCOS中，采用多级预测方法可以有效地解决模糊问题，与基于锚框的模糊检测器相比，基于模糊控制器的模糊检测器具有更好的性能

前面提到，为了解决真实边框重叠带来的模糊性和低召回率，FCOS采用类似FPN中的多级检测，就是在不同级别的特征层检测不同尺寸的目标。

FCOS通过直接限定不同特征级别的边界框的回归范围来进行分配

此外，FCOS在不同的特征层之间共享信息，不仅使检测器的参数效率更高，而且提高了检测性能。

### Center-ness

<center><img alt="center-ness" src="https://pic2.zhimg.com/80/v2-6300c9570dcb7196aa07a012443345bd_720w.jpg"/></center>

通过多级预测之后发现FCOS和基于锚框的检测器之间仍然存在着一定的距离，主要原因是距离目标中心较远的位置产生很多低质量的预测边框。

在FCOS中提出了一种简单而有效的策略来抑制这些低质量的预测边界框，而且不引入任何超参数。具体来说，FCOS添加单层分支，与分类分支并行，以预测"Center-ness"位置。

$$centerness^*=\sqrt{\frac{min(l^*, r^*)}{max(l^*, r^*)}\times\frac{min(t^*, b^*)}{max(t^*, b^*)}}$$

center-ness(可以理解为一种具有度量作用的概念，在这里称之为"中心度")，中心度取值为0,1之间，使用交叉熵损失进行训练。并把损失加入前面提到的损失函数中。测试时，将预测的中心度与相应的分类分数相乘，计算最终得分(用于对检测到的边界框进行排序)。因此，中心度可以降低远离对象中心的边界框的权重。因此，这些低质量边界框很可能被最终的非最大抑制（NMS）过程滤除，从而显着提高了检测性能。

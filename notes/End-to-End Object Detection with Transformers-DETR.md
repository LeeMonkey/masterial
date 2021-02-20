# End-to-End Object Detection with Transformers-DETR

Github开源地址:[https://link.zhihu.com/?target=https%3A//github.com/facebookresearch/detr](https://link.zhihu.com/?target=https%3A//github.com/facebookresearch/detr)

## 1. 创新点

将目标检测任务转化成一个序列预测（set prediction）任务，使用transformer编码-解码器结构和双边匹配的方法，由输入图像直接得到预测结果序列。没有proposal（Faster R-CNN），没有anchor（YOLO、SSD），没有center（CenterNet），没有NMS，直接预测检测框和类别，利用二分图匹配的匈牙利算法，将CNN和transformer巧妙的结合，实现目标检测的任务。

<center><img alt="DETR整体结构" src="https://pic1.zhimg.com/v2-477a4e2a04b4913e1d8dd4b67e4df0f0_r.jpg"/><br/>DETR整体结构</center>

本检测框架两个重要因素：

- 使预测框和ground truth之间一对一匹配的序列预测loss
- 预测一组目标序列，并对它们之间关系进行建模的网络结构。接下来依次介绍这两个因素的设计方法。

### 1.1 模型的整体结构

<center><img alt="DETR整体结构" src="https://pic1.zhimg.com/v2-aae1329060cd9d50df17c4e7a421e09c_r.jpg"/></center>

按作用：Backbone + Transformer + Prediction
按结构：CNN + Encoder + Decoder + FFN

- Backbone

输入：$3 \times W_0 \times H_0$ ==> 输出：$2048 \times \frac{W_0}{32} \times \frac{H_0}{32}$

- Transformer

<center><img alt="Transformer" src="https://pic2.zhimg.com/v2-1be61511d53dca07f1c83697eb23a87d_r.jpg"/></center>

Transformer encode部分将输入的特征图降维并flatten，然后送入下图左半部分所示的结构中，和空间位置编码一起并行经过多个自注意力分支、正则化和FFN，得到一组长度为N的预测目标序列。  

接着，将Transformer encoder得到的预测目标序列经过上图右半部分所示的Transformer decoder，并行的解码得到输出序列（而不是像机器翻译那样逐个元素输出）。和传统的autogreesive机制不同，每个层可以解码N个目标，由于解码器的位置不变性，即调换输入顺序结果也不会变化，除了每个像素本身的信息，位置信息也很重要，所以这N个输入嵌入必须不同以产生不同的结果，所以学习NLP里面的方法，加入positional encoding并且每层都加，**作者非常用力的在处理position的问题，在使用 transformer 处理图片类的输入的时候，一定要注意position的问题**。

- Prediction

使用共享参数的FFNs（由一个具有ReLU激活函数和d维隐藏层的3层感知器和一个线性投影层构成）独立解码为包含类别得分和预测框坐标的最终检测结果（N个），FFN预测框的标准化中心坐标，高度和宽度w.r.t. 输入图像，然后线性层使用softmax函数预测类标签。

<center><img src="https://pic2.zhimg.com/v2-e494531de0ddd5536a8c9f728e86c7ad_r.jpg"/><br/><img src="https://pic2.zhimg.com/v2-0cf0f31b74cda6e84aefdad0d134f235_r.jpg"/></center>

## 2. 模型损失函数

基于序列预测的思想，作者将网络的预测结果看作一个长度为N的固定顺序序列$\tilde{y}$，$\tilde{y} = \tilde{y_i}, i \in (1, N)$，（其中N值固定，且远大于图中ground truth目标的数量）$\tilde{y_i} = (\tilde{c_i}, \tilde{b_i})$，同时将ground truth也看作一个序列$y: y_i = (c_i, b_i)$（长度一定不足N，所以用$\phi$（表示无对象）对该序列进行填充，可理解为背景类别，使其长度等于N），其中$c_i$表示该目标所属真实类别，$b_i$表示为一个四元组（含目标框的中心点坐标和宽高，且均为相对图像的比例坐标）。

那么预测任务就可以看作是$y$与$\tilde{y}$之间的二分图匹配问题，采用匈牙利算法[<sup>[1]</sup>](#refer)作为二分匹配算法的求解方法，定义最小匹配的策略如下：

$$\hat{\sigma} = \underset{\sigma \in \sigma_N}{argmin}\sum^{N}_{i}{\mathcal{L_{match}(y_i,\hat{y_{\sigma(i)}})}}$$

求出最小损失时的匹配策略$\tilde{\sigma}$，对于$\mathcal{L}_{match}$同时考虑了类别预测损失即真实框之间的相似度预测。

对于$\sigma(i)$，$c_i$的预测类别置信度为$\tilde{P}_{\sigma(i)}(c_i)$，边界框预测为$\tilde{b}_{\sigma(i)}$，对于非空的匹配，定于$\mathcal{L}_{match}$为：

$$-\mathbb{I}_{\{ c_i \neq \varnothing \hat{P}_{\sigma(i)} \}}(c_i) + \mathbb{I}_{\{c_i \neq \varnothing \}}\mathcal{L}_{box}(b_i, \hat{b}_{\sigma(i)})$$

进而得出整体的损失：

$$\mathcal{L}_{Hungarian}(y,\hat{y}) = \sum^N_{i = 1} \left[ - log\hat{p}_{\hat{\sigma}(i)}(c_i) + \mathbb{I}_{\{ c_i \neq \varnothing\}} \mathcal{L}_{box} (b_i, \hat{b}_{\sigma}(i)) \right]$$

考虑到尺度的问题，将L1损失和IoU损失线性组合，得出$L_{box}$如下：

$$\lambda_{iou} \mathcal{L}_{iou}(b_i, \hat{b}_{\sigma(i)}) + \lambda_{L_1}\| b_i - \hat{b}_{\sigma(i)}\|_1$$

$L_{box}$采用的是Generalized intersection over union论文提出的GIOU[<sup>[2]</sup>](#refer),关于GIOU后面会大致介绍。

为了展示DETR的扩展应用能力，作者还简单设计了一个基于DETR的全景分割框架，结构如下：

<center><img src="https://pic1.zhimg.com/80/v2-2b9fad8f3430b22f47251fd62394f108_720w.jpg"/></center>

## 附录

GIOU：

<center><img src="https://pic3.zhimg.com/80/v2-7850a530d8b59e9a25f3e9ed80b841d2_720w.jpg"/><br/><img src="https://pic1.zhimg.com/80/v2-b331f280135dbfc9c6b036d46ca3d58c_720w.jpg"/></center>

$$L_{GIoU} =  1- GIoU$$

- 与IoU相似，GIoU也是一种距离度量，作为损失函数的话，满足损失函数的基本要求
- GIoU对scale不敏感
- GIoU是IoU的下界，在两个框无线重合的情况下，IoU=GIoU
- IoU取值[0,1]，但GIoU有对称区间，取值范围[-1,1]。在两者重合的时候取最大值1，在两者无交集且无限远的时候取最小值-1，因此GIoU是一个非常好的距离度量指标。
- 与IoU只关注重叠区域不同，GIoU不仅关注重叠区域，还关注其他的非重合区域，能更好的反映两者的重合度。

---

<div id="refer"></div>

[1] [匈牙利算法](https://baike.baidu.com/item/%E5%8C%88%E7%89%99%E5%88%A9%E7%AE%97%E6%B3%95/9089246?fr=aladdin#refer1)  
[2] [Generalized intersection over union](http://openaccess.thecvf.com/content_CVPR_2019/html/Rezatofighi_Generalized_Intersection_Over_Union_A_Metric_and_a_Loss_for_CVPR_2019_paper.html)

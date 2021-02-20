# DCANet

## 导读

DCANet区别于其他Attention方法在于该方法是用来提升Attention模块的能力，主要做法是：将相邻的Attention Block互相连接，让信息在Attention模块之间流动。

## 原理

**Attention机制**：探索特征之间的依赖关系来得到更好的特征表示。**Attention机制分为三部分：通道Attetion、空间Attention和self-attention**，self-attention可以看为通道和空间注意力的混合。

**残差链接**：例如ResNet、ResNext、DenseNet，残差链接使网络变宽，缓解网络退化的情况。

**连接注意力**：RA-CNN是专门用于细粒度图像识别的网络架构，网络可以不断生成具有鉴别力的区域，从而实现从粗糙到细节识别。GANet中，高层的Attention特征会送往底层的特征来指导注意力学习。

## 主要部分

Attention模块设计分成三个阶段，分别是：

- Extraction：特征提取阶段
- Transformation：转换阶段，将信息进行处理或融合
- Fusion：融合阶段，将得到的信息进行融合到主分支中

下图从左到右分别是SEblock、CEblock、GCblock、SKblock：
<center><img src="https://mmbiz.qpic.cn/mmbiz_png/KvFmdiaVWqQlJm1oHGG3k9RSeiaSTjWbRsCajzwM4MFSp60gbiagI0nNt9Liclgn9XkKugHMLxS69MFnLFdg16Zib6Q/640"/></center>
<center>通常范式</center>

### Extraction

从特征图上提取特征，特征图为：$X\in{R^{C\times{H}\times{W}}}$经过特征提取器$g$（$\omega_{g}$是特征提取操作的参数，$G$是输出结果）：$G=g(X,\omega_{g})$

### Transformation

这个阶段处理上一个阶段得到的聚集信息，然后将他们转化到非线性注意力空间中。规定转换t为（$\omega_{g}$是参数，$T$是这个阶段的输出结果）：
$$T=t(G,\omega_{t})$$

### Fusion

整合上个阶段获取的特征图：
$$X^{'}_{i}=T_{i}\circledast{X_{i}}$$

$X^{'}_{i}=T_{i}$代表最终这一层的输出结果，$\circledast$代表特征融合方式，比如点乘、求和等操作

### Attention Connection

特征不仅要从当前层获取，还要从上一层获取，需要融合两层的信息，具体融合方式有两种：

- Direct Connection:通过相加的方式相连。
$$f(\alpha G_i,\beta\tilde{T}_i)=\alpha G_i + \beta\tilde{T}_i$$

- Weighted Connection：加权的方式进行相连。
$$f(\alpha G_i,\beta\tilde{T}_i)=\frac{{\lvert \alpha G_i\rvert}^{2}}{\alpha G_i + \beta\tilde{T}_i} + \frac{{\lvert \beta\tilde{T}_i \rvert}^{2}}{\alpha G_i + \beta\tilde{T}_i}$$

<center><img src="https://mmbiz.qpic.cn/mmbiz_png/KvFmdiaVWqQlJm1oHGG3k9RSeiaSTjWbRs2Z0icxQ71mqNkL83w9kWUTJSlHVYTiaqYXHENFuxDq1KvRVb7DkaXDtA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1"/><br/>DCANet示意图</center>
上面通路是两个通道注意力模块，两个模块之间会有一个Attention Connection让两者相连，后面的模块能利用之前的模块信息。下边通路是两个空间注意力模块，原理同上，空间注意力模块可以有也可以没有。

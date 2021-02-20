# 详解Transformer

## 前言

Transformer中抛弃了传统的CNN和RNN，整个网络结构完全是由Attention机制组成。更进一步说，Transformer是由且仅由self-Attention和Feed Forward Neural Network组成。

Transformer的提出解决了两个问题，使用Attention机制，将序列中的任意两个位置之间的距离缩小为一个常量。其次它不像RNN这样的顺序结构，因此具有更好的并行性。

论文中给出的Transformer的定义：Transformer is the first transduction model relying entirely on self-attention to compute representations of its input and output without using sequence aligned RNNs or convolution。

## 1. Transformer详解

### 1.1 高层Transformer

Transformer本质上是一个Encoder-Decoder的结构，可以表示为下图的结构：

<center><img alt="Transformer的Encoder-Decoder结构" src="https://pic1.zhimg.com/v2-5a252caa82f87920eadea2a2e93dc528_r.jpg"/><br/>Transformer的Encoder-Decoder结构</center>

如论文中所设置的，编码器由6个编码block组成，同样解码器是6个解码block组成。与所有的生成器模型相同的是，编码器的输出会作为解码器的输入，如下图所示：

<center><img alt="Transformer的Encoder和Decoder均由6个block堆叠而成" src="https://pic3.zhimg.com/80/v2-c14a98dbcb1a7f6f2d18cf9a1f591be6_720w.jpg"/><br/>Transformer的Encoder和Decoder均由6个block堆叠而成</center>

在Transformer的encoder中，数据首先会经过一个叫做self-attention的模块得到一个加权之后的特征向量 $Z$，这个$Z$便是论文公式1中的$Attention(Q,K,V)$:

$$Attention(Q,K,V) = softmax(\frac{QK^T}{\sqrt{d_k}})V\tag{1}$$

得到$Z$之后，它会被送到encoder的下一个模块，即Feed Froward Neural Network。这个全连接有两层，第一层 的激活函数是ReLU，第二层是一个线性激活函数，可以表示为：

$$FNN(Z) = max(0,ZW_1 + b_1)W_2 + b_2\tag{2}$$

Encoder的结构如下图所示：

<center><img alt="Transformer由self-attention和Feed Forward neural network组成
" src="https://pic3.zhimg.com/80/v2-89e5443635d7e9a74ff0b4b0a6f31802_720w.jpg"/><br/>Transformer由self-attention和Feed Forward neural network组成
</center>

Decoder的结构如下图所示，他和Encoder的不同之处在于Decoder多了一个Encoder-Decoder Attention，两个Attention分别用于计算输入和输出的权值：

1. Self-Attention：当前翻译和已经翻译的前文之间的关系
2. Encoder-Decoder Attention：当前翻译和编码的特征向量之间的关系

<center><img alt="Transformer的解码器由self-attention，encoder-decoder attention以及FFNN组成" src="https://pic1.zhimg.com/80/v2-d5777da2a84e120846c825ff9ca95a68_720w.jpg"/><br/>Transformer的解码器由self-attention，encoder-decoder attention以及FFNN组成</center>

### 1.2 输入编码

1.1节介绍的就是Transformer的主要框架，下面我们将介绍它的输入数据。如下图所示，首先通过Word2Vec等词嵌入方法将输入语料转换成特征向量，论文中使用的词嵌入的维度为$d_{model}=512$。

<center><img alt="单词的输入编码" src="https://pic1.zhimg.com/v2-408fcd9ca9a65fdbf9d971cfd9227904_r.jpg"/><br/>单词的输入编码</center>

在最底层的block中，$x$将直接作为Transformer的输入，而在其他层中，输入则是上一个block的输出。如下图所示：

<center><img alt="输入编码作为一个tensor输入到encoder中" src="https://pic2.zhimg.com/80/v2-ff0f90ebee18dd909999bd3bee38fa45_720w.jpg"/><br/>输入编码作为一个tensor输入到encoder中</center>

### 1.3 Self-Attention

Self-Attention是Transformer最核心的内容，然而作者并没有详细讲解，下面我们来补充作者遗漏的地方。回想Bahdanau等人突出的用Attention[<sup>[1]</sup>](#refer1),其核心内容是为输入向量的每个单词学习一个权重，例如在下面的例子中我们判断it的代指内容

```text
The animal didn't cross the street because it was too tired
```

通过加权后可以得到类似与下图的加权情况，在讲解self-attention的时候我们也会使用下图类似的表示方式：

<center><img alt="经典Attention可视化示例图" src="https://pic2.zhimg.com/80/v2-d2129d06290744ebc12b6f220866b2a5_720w.jpg"/><br/>经典Attention可视化示例图</center>

在self-attention中，每个单词有3个不同的向量，它们分别是Query向量（$Q$），Key向量（$K$）和Value向量（$V$），长度均为64。它们是通过3个不同的权值矩阵由嵌入向量$X$乘以三个不同的权值矩阵$W^Q$， $W^K$，$W^V$得到，其中三个矩阵的尺寸也是相同的。均是$512 \times 64$。

<center><img alt="Q,K,V的计算示例图" src="https://pic2.zhimg.com/80/v2-159cd31e629170e0bade136b91c9de61_720w.jpg"/><br/>Q, K, V的计算示例图</center>

Attention的计算方法可以分为7步：

1. 如上文，将输入单词转化成嵌入向量
2. 根据嵌入向量得到$q$，$k$，$v$
3. 为了每个向量计算一个score：$score=q*k$
4. 为了梯度的稳定，Transformer使用了score归一化，即除以$\sqrt{d_k}$
5. 对score施以softmax激活函数
6. softmax点成Value值$v$，得到加权的每个输入向量的评分$v$
7. 相加之后得到最终的输出结果$z$：$z = \sum{v}$

上面步骤可以表示为下图的形式：

<center><img alt="Self-Attention计算示例图" src="https://pic1.zhimg.com/80/v2-79b6b3c14439219777144668a008355c_720w.jpg"/><br/>Self-Attention计算示例图</center>

实际计算过程中是采用基于矩阵的计算方式，那么论文中的$Q$，$V$，$K$的计算方式如下图：

<center><img alt="Q，V，K的矩阵表示" src="https://pic3.zhimg.com/80/v2-bcd0d108a5b52a991d5d5b5b74d365c6_720w.jpg"/><br/>Q，V，K的矩阵表示</center>

最终总结为下图所示的矩阵形式：

<center><img alt="Self-Attention的矩阵表示" src="https://pic1.zhimg.com/80/v2-be73ba876922cf52df8a00a55f770284_720w.jpg"/></center>

这里也就是公式1的计算方式。

在self-attention需要强调的最后一点是其采用了残差网络[<sup>[2]</sup>](#refer2)中的short-cut结构，目的当然是解决深度学习中的退化问题，得到的最终结果如下图所示。

<center><img alt="Self-Attention中的short-cut连接" src="https://pic1.zhimg.com/80/v2-2f06746893477aec8af0c9c3ca1c6c14_720w.jpg"/><br/>Self-Attention中的short-cut连接</center>

### 1.3 Multi-Head Attention

Multi-Head Attention相当于$h$个不同的self-attention的集成（ensemble），在这里我们以$h = 8$举例说明。Multi-Head Attention的输出分成3步：

1. 将数据$X$分别输入到上图所示的8个self-attention中，得到8个加权后的特征矩阵$Z_i, i\in\{1,2,...,8\}$。
2. 将8个$Z_i$按列拼成一个大的特征矩阵
3. 特征矩阵经过一层全连接后得到输出$Z$。

整个过程如下图所示：

<center><img alt="Multi-Head Attention" src="https://pic3.zhimg.com/80/v2-c2a91ac08b34e73c7f4b415ce823840e_720w.jpg"/><br/>Multi-Head Attention</center>

同self-attention一样，multi-head attention也加入了short-cut机制。

### 1.4 Encoder-Decoder Attention

在解码器中，Transformer block比编码器中多了个encoder-decoder attention。在encoder-decoder attention中，$Q$来自解码器的上一个输出，$K$和$V$则来自于解码器的输出。起计算方式完全和1.3节中Attention的计算方式相同。

由于在机器翻译中，解码过程是一个顺序操作的过程，也就是当解码第K个特征向量时，我们只能看到第k-1及其之前的解码结果，论文中把这种情况下的multi-head attention叫做masked multi-head attention。

### 1.5 损失层

解码器解码后，解码的特征向量经过激活函数为softmax的全连接之后得到反映每个单词概率的输出向量。此时我们便可以通过CTC等损失函数训练模型。

而一个完整可训练的网络结构便是encoder和decoder的堆叠（各$N$个，$N=6$），便得到下图中完整的Transformer的结构：

<center><img alt="Transformer的完整结构" src="https://pic1.zhimg.com/80/v2-9fb280eb2a69baf5ceafcfa3581aa580_720w.jpg"/><br/>Transformer的完整结构</center>

## 2. 位置编码

截止目前为止，我们介绍的Transformer模型并没有捕捉顺序序列的能力，也就是说无论句子的结构怎么打乱，Transformer都会得到类似的结果。换句话说，Transformer只是一个功能更强大的词袋模型而已。

为了解决这个问题，论文中在编码词向量时引入了位置编码（Position Embedding）的特征。具体地说，位置编码会在词向量中加入了单词的位置信息，这样Transformer就能区分不同位置的单词了。

那么怎么编码这个位置信息呢？常见的模式有：a. 根据数据学习；b. 自己设计编码规则。在这里作者采用了第二种方式。那么这个位置编码该是什么样子呢？通常位置编码是一个长度为$d_{model}$的特征向量，这样便于和词向量进行单位加的操作，如下图所示：

<center><img alt="Position Enbedding" src="https://pic3.zhimg.com/80/v2-3e1304e3f8da6cf23cc43ab3927d700e_720w.jpg"/><br/>Position Embedding</center>

编码公式如下：

$$PE(pos, 2i)=sin(\frac{pos}{10000^{\frac{2i}{d_{model}}}})\tag{3}$$  

$$PE(pos, 2i + 1) = cos(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\tag{4})$$

在上式中，$pos$表示单词的位置，$i$表示单词的维度。关于位置编码的实现可在Google开源的算法中[get_timing_signal_1d()](https://link.zhihu.com/?target=https%3A//github.com/tensorflow/tensor2tensor/blob/23bd23b9830059fbc349381b70d9429b5c40a139/tensor2tensor/layers/common_attention.py)函数找到对应的代码。

作者这么设计的原因是考虑到在NLP任务重，除了单词的绝对位置，单词的相对位置也非常重要。根据公式$sin(\alpha + \beta) = sin{\alpha}cos{\beta} + cos{\alpha}sin{\beta}$以及$cos(\alpha + \beta) = cos{\alpha}cos{\beta} - sin{\alpha}sin{\beta}$，这表明位置$k + p$的位置向量可以表示为位置$k$的特征向量的线性变化，这为模型捕捉单词之间的相对位置关系提供了非常大的便利。

## 3. 总结

优点：（1）虽然Transformer最终也没有逃脱传统学习的套路，Transformer也只是一个全连接（或者是一维卷积）加Attention的结合体。但是其设计已经足够有创新，因为其抛弃了在NLP中最根本的RNN或者CNN并且取得了非常不错的效果，算法的设计非常精彩，值得每个深度学习的相关人员仔细研究和品位。（2）Transformer的设计最大的带来性能提升的关键是将任意两个单词的距离是1，这对解决NLP中棘手的长期依赖问题是非常有效的。（3）Transformer不仅仅可以应用在NLP的机器翻译领域，甚至可以不局限于NLP领域，是非常有科研潜力的一个方向。（4）算法的并行性非常好，符合目前的硬件（主要指GPU）环境。

缺点：（1）粗暴的抛弃RNN和CNN虽然非常炫技，但是它也使模型丧失了捕捉局部特征的能力，RNN + CNN + Transformer的结合可能会带来更好的效果。（2）Transformer失去的位置信息其实在NLP中非常重要，而论文中在特征向量中加入Position Embedding也只是一个权宜之计，并没有改变Transformer结构上的固有缺陷。

---

## Reference

<div id="refer1"></div>
[1] Bahdanau D, Cho K, Bengio Y. Neural machine translation by jointly learning to align and translate[J]. arXiv preprint arXiv:1409.0473, 2014.
<div id="refer2"></div>
[2] He K, Zhang X, Ren S, et al. Deep residual learning for image recognition[C]//Proceedings of the IEEE conference on computer vision and pattern recognition. 2016: 770-778.

# Training ImageNet in 1 Hour

## 1. 步长

在[Asynchronous Parallel Stochastic Gradient for Nonconvex Optimization, In NIPS 2015](https://arxiv.org/abs/1506.08272)一文中指出，SGD的步长（learning rate）应该选择为$\gamma = \frac{1}{\sqrt{MK\sigma^2}}$量级才能保证$\frac{1}{\sqrt{MK}}$量级的收敛速度（也就是有并行加速），其中M为batch size，K为iteration数，$\sigma^2$为stochastic gradient的方差上界，因为MK就是一共使用的sample个数，故在training同样多的epodh的情况下，MK为定值，所以步长不变。
之所以不变，是因为在我们的paper里面使用的update rule是：

$$x_{k+1} \leftarrow x_k - \gamma \sum^M_{i=1}i-th \quad stochastic\quad gradient$$

而Facebook paper中是：

$$x_{k+1} \leftarrow x_k - \gamma' \frac{1}{M}\sum^M_{i=1}i-th \quad stochastic\quad gradient$$

也就是说他们的stochastic gradient是被平均过，**如果要保证新的步长与原来等价，就要保证$\gamma = \frac{\gamma'}{M}$, 所以$\gamma'$就要随batch size（M）线性增长了**。

下面吗要将facebook这篇文章没有的insight。从

$$x_{k+1} \leftarrow x_k - \gamma' \frac{1}{M}\sum^M_{i=1}i-th \quad stochastic\quad gradient$$

的步长线性增长我们实际上可以看出这种做法是让每个stochastic gradient系数不变，也就是跟batch size小的时候发挥一样的作用，那么为什么batch size加太大会失败。因为这些stochastic gradient的平均当成一个gradient，那对于这个平均过后的gradient的步长为$M_\gamma$。在batch size很大的时候，这个平均相当接近true gradient，如果用gradient descent，步长应该为$\frac{1}{L}$，其中L为客观存在的gradient Lipshitz constant，也就是gradient descent步长是常数，既然batch size很大的时候stochastic gradient descent趋近于gradient descent，那么步长就不能任意线性增大，之前我们说的$\gamma' = \frac{M}{\sqrt{MK\sigma^2}}$实际上实在K远大于M时候的结论。在一般情况下步长应该取$\frac{1}{L + \sqrt{\frac{K\sigma^2}{M}}}$在M很大时，这个步长就变成了$\frac{1}{L}$，退化为gradient descent步长。**所以说步长随batch size 线性增长也不准确，确切说时逐渐增长到定值**（在[A Comprehensive Linear Speedup Analysis for Asynchronous Stochastic Parallel Optimization from Zeroth-Order to First-Order, In NIPS 2016](https://arxiv.org/abs/1606.00498)有说明，当然我们这个还是在搞Async算法，sync的只是async的一种特殊情况）。

## Warm Up

warmup这个其实取决于数据集，前面我们都是把stochastic gradient的variance $\sigma^2$当作常数，但实际中一开始离解的距离比较远的时候，stochastic gradient的大小和方差可能很大（取决于model和dataset），而逐渐接近解的时候，方差会逐渐减小，于是就会出现一开始取小一些的步长的做法。

## 分布式SGD

每个iteration所有黑色节点从parameter server拿到模型weights, 然后计算gradient, 再把gradient push回 parameter server。parameter server将这些gradient加起来update到parameter server保存的weights上去。 这样就完成了一个iteration。这样相当于把一个大batch分成很多份让很多黑色节点一起算。但是问题是如果有很多节点的时候，每个节点都要和parameter server通讯，对parameter server带宽压力很大。系统拓扑图如下所示：

<center><img alt="" src="https://pic1.zhimg.com/50/v2-44f43aff4706e1d08e30a468cad0083a_hd.jpg"/></center>

异步并行的SGD缓解了这个问题（比如我之前说的两个paper），因为异步SGD每个节点可以无脑从parameter server 取weights以及无脑push gradient，不需要同步，这样各个节点跟parameter server的通讯可以错开。在异步的情况下，可以证明在节点数限制住的情况下，可以有跟同步SGD一样的收敛速度。(还是见我前面说的俩paper)。

但如果节点数继续增多，parameter server还是会有很大压力。这时候我们可以用AllReduce方法去掉中心的 parameter server做同步并行SGD。AllReduce具体过程描述起来比较麻烦，但大体意思是每个节点只跟它相邻的节点通讯（比如把所有节点连成一个环)）。这样通讯就不再集中于某个parameter server，而是被所有节点分担了。但AllReduce有一个问题，如果网络有n个节点，则AllReduce要将每份gradient切成n份再一份一份传，这样会导致网络 latency比较大的时候, 延迟对整体收敛速度的影响会非常大。

[Can Decentralized Algorithms Outperform Centralized Algorithms? A Case Study for Decentralized Parallel Stochastic Gradient Descent](https://arxiv.org/abs/1705.09056)讲了在分布式SGD中，比如topology可以是：

<center><img src="https://pic1.zhimg.com/80/v2-4379d7d458eda03966987f6d90188275_720w.jpg"/></center>

没有中心的parameter server。每个节点local keep一份weights的复制。每个节点只需要从它的相邻节点把他们 local的weights拿来，跟自己的average，然后将自己local算出的gradient update到average过的新weights上。这篇文章**证明了**在这种情况下整个网络的local weights的平均依然收敛，而且收敛速度与同步并行SGD一样。也就是说**有加速**。而且在这种情况下，每个iteration每个节点只需要跟近邻通讯一次，而不是AllReduce中的n次。这样大幅减少了communication上的问题，可以最大幅度地scale。

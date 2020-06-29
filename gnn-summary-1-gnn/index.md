# 图神经网络总结(一) —— 图神经网络


最近在看图神经网络的综述，数学功底太差看的云里雾里，记录一下学习过程中接触到的一些概念和自己的理解。

## 0 简介

图像、语音、文本等都可以表示为欧几里得结构数据，传统的神经网络如CNN、RNN在处理这些欧几里得结构数据上取得了不错的效果。关于欧几里得数据和非欧几里得数据，引用论文 [Geometric deep learning: going beyond Euclidean data](https://arxiv.org/abs/1611.08097) 中的一段话：

> `Many scientific fields study data with an underlying structure that is a non-Euclidean space.` Some examples include social networks in computational social sciences, sensor networks in communications, functional networks in brain imaging, regulatory networks in genetics, and meshed surfaces in computer graphics. In many applications, such geometric data are large and complex (in the case of social networks, on the scale of billions), and are natural targets for machine learning techniques. In particular, we would like to use deep neural networks, which have recently proven to be powerful tools for a broad range of problems from computer vision, natural language processing, and audio analysis. `However, these tools have been most successful on data with an underlying Euclidean or grid-like structure, and in cases where the invariances of these structures are built into networks used to model them.`

> In image analysis applications, one can consider  images as functions on the Euclidean space (plane),  sampled on a grid. 

原文把非欧几里得数据定义为数据潜在的结构是非欧空间的。以图像为例，我们可以看做一个欧氏平面（二维），在一个网格上采样得到图像的数据。（个人认为其实就是数据以什么方式表示的问题）

显然社交网络、知识图谱一类的图数据没办法直接用传统的神经网络处理，因此我们可以通过图嵌入(Graph Embedding)获得结点、结构等信息的向量表示，再应用到传统的机器学习方法上，而图神经网络相当于直接在图数据上进行学习，完成各种任务。图嵌入和图神经网络是密切相关也是非常相似的两个领域。

CNN: Extract Spatial Features

## 1 图神经网络

图神经网络(Graph Neural Networks, GNN)最早在[A new model for learning in graph domains(2005年)](https://ieeexplore.ieee.org/document/1555942)提出，相关的工作还有[The Graph Neural Network Model(2009年)](https://ieeexplore.ieee.org/document/4700287)，基本的思想是通过邻居结点间传播信息，获得图或结点的表示，这里介绍的图神经网络使用简单的前馈神经网络聚合邻居结点的信息。

### 1.1 巴拿赫不动点定理

巴拿赫不动点定理又称压缩映射定理。

令 $(X, d)$ 为非空的完备度量空间，若映射 $f:X \rightarrow X$ 满足 $\forall x, y \in X, d(x, y) \le \lambda d(T(x), T(y))$ ，则称 $f$ 是一个压缩映射。

巴拿赫不动点定理是指在完备度量空间中 $(X, d)$ ，设 $f:X \rightarrow X$ 是一个压缩映射，则$f$在$X$中存在唯一不动点，对于 $\forall x \in X$ ,序列 $x, f(x), f(f(x)) ...$ 收敛至 $f$ 的不动点。

### 1.2 模型细节

GNN定义两个函数：$f$ 是局部转移函数(local transaction function)，用于整合自身结点、邻居结点以及边的信息：

$$ h_v = f(x_v, x_{co[v]}, h_{ne[v]}, x_{ne[v]}) \tag{1.1} $$

其中 $h_v, x_v, x_{co[v]}, h_{ne[v]}, x_{ne[v]}$ 分别代表结点v的状态，结点v的特征，结点v的边的特征，邻居结点的状态，邻居结点的特征。

$g$ 是局部输出函数(local output function)：

$$ o_v = g(x_v, h_v) \tag{1.2} $$

$f$ 和 $g$都可以看做一个前馈神经网络，对于每个结点都是共享的，同时还要限制函数 $f$ 为压缩映射。 

上面是每个结点状态和输出的函数，现在根据整个图整合成更紧凑的表达形式：

$$ H = F(H, X) \tag{1.3} $$
$$ O = G(H, X_v) \tag{1.4} $$

$H, X_v$ 分别由所有结点的状态，所有结点的特征堆叠而成。

注： $H^{(t)} = F(H^{(t-1)}, X)$ ， $F$ 是关于 $H$ 的压缩映射。在计算所有结点的状态 $H^{(t-1)}$ 后，每个结点 $v$ 的邻居结点 $ne[v]$ 也接收到了其邻居结点的信息，因此不断迭代计算 $H^{(t)}, H^{(t+1)} ...$可以接收到更远的结点的信息，不动点定理实际上保证了解的存在（不至于无穷无尽的迭代下去）。 

最后是损失函数的部分。假设图中 $p$ 个结点拥有监督信息，则损失函数的形式为：

$$ loss = \sum_{i=1}^{p} (t_i - o_i) \tag{1.5} $$

### 1.3 训练过程

注意训练过程每个step都需要初始化每个结点的状态 $h_v$ 。

1. 初始化H(0), 迭代计算$H^{(t)} = F(H^{(t-1)}, X)$直至稳定，计算loss
2. 根据loss计算梯度，更新参数
3. 重复1、2直至收敛

关于梯度的计算可以参考原论文 [The Graph Neural Network Model](https://ieeexplore.ieee.org/document/4700287)

### 1.4 模型的局限

1. 显然，基于不动点定理会使结点的状态比较相似，即Over Smooth，对于node-focused任务效果可能比较差
2. 结点状态迭代的效率问题
3. 部分边的信息无法被很好的建模，比如边的类型

## 2 图神经网络的变种 

### 2.1 门控图神经网络(Gated Graph Neural Networks)

GG-NN整体结构和GNN比较相似，最大的区别在于局部转移函数使用门控循环单元(GRU)代替，而GNN中局部转移函数是普通的前馈神经网络(FNN)，并且状态的转移不再依赖于不动点定理，而是迭代固定的次数 $T$



## 参考

[[1]. 从图(Graph)到图卷积(Graph Convolution)：漫谈图神经网络模型 (一)](https://www.cnblogs.com/SivilTaram/p/graph_neural_network_1.html)

[[2]. A Comprehensive Survey on Graph Neural Networks](https://arxiv.org/abs/1901.00596)

[[3]. Geometric deep learning: going beyond Euclidean data](https://arxiv.org/abs/1611.08097)

[[4]. Graph Neural Networks: A Review of Methods and Applications](https://arxiv.org/abs/1812.08434)

[[5]. The Graph Neural Network Model](https://ieeexplore.ieee.org/document/4700287)

[[6]. Gated Graph Sequence Neural Networks](https://arxiv.org/abs/1511.05493)



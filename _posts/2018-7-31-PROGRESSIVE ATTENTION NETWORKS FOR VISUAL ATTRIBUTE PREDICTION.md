* 1. Introduction

两种视觉注意力机制：1.根据任务取特征子集，这部分特征对应着输入图像的某块区域，即感兴趣区域。soft attention和本文都属于着一种。
2.通过图像进行变换。通过训练出的变换关系得到输出和输入之间的映射关系（采样表）对特征图相应位置进行采样得到ROI。如spatial transform network.

* 2. network

(/images/attention1_1.PNG)

本文叫做渐进注意力机制，是多层的soft attention。在每一层通过attention网络根据输入特征和查询得到attention probility map，对应特征图每个像素点
是否与任务相关的概率值，然后将attention map和特征图相乘得到attended feature map，也即下一阶段卷积层的输入。它逐步抑制不相关区域最后将相关区域凸现出来。如图

(/images/attention1_2.PNG)

(/images/attention1_3.PNG)

$g_att(f,q;\theta)$是注意力网络的函数表示，根据当前特征图某个位置的特征f和查询（标签）q得出该点与任务相关性的大小，$\theta$是网络参数。将得出的值送入softmax归一化得到权重值。然后权重和特征图相乘获得过滤后的attended特征图，$\alpha^l_{i,j} * f^l_{i,j}$。最后一层的时候对所有像素点求和得到1x1xC的向量。$\sum_{i}^{W}\sum_{j}^{H}\alpha^l_{i,j}*f^L_{i,j}$。

* 2.1 attention network

(/images/attention1_4.PNG)

注意力网络使用了像素点的上下文信息，论文中是在宽高方向各扩种两个像素点，然后把该点处的向量和标签向量（one-hot格式）组合送入后面的两层全连接层得出相关性。

作者人为在pool出特征图才会降低尺寸，所以注意力层只位于每个stage后面。

* Experiment

(/images/attention1_5.PNG)

上述三个数据集分别是原生mnist，对数字做尺度缩放，将数字混合到自然场景下。可以看出STN在复杂背景很难提取有效地ROI，从PAN-CTX和PAN（未使用上下文）看出在自然场景下识别上下文信息会带来很大帮助。还有两种分别是soft attention 和hard attention，这两种attention都只作用于最后一层

注意力机制可有效的应用与分类任务中。
(/images/attention1_6.PNG)

b,c分别是attention map，attented map，a是体现在原生图像上的效果。自左向右分别是三层注意力层可以看出效果逐步变好。作者没有在第一二个pool后用PAN是觉得这两层特征不够区分性可能是可视化之后考虑的。

(/images/attention1_7.PNG)

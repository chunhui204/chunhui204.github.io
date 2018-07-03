---
layout: post
title: "【arXiv2016】Scene Text Detection via Holistic, Multi-Channel Prediction"
date: 2018-7-3
tag: text detection,  FCN
---   
这是将文本检测看做语义分割问题的第三篇（前两篇是text detection with FCN和cascade CNN），模型运行在整张图上产生全局的像素及预测。本文提取了文本的三种特征：文本块特征、字符区域和两者关系，和text detection with FCN相似，只是改进了字符区域聚合的方法。

本文思路：使用一个网络产生文本块，字符区域和连接方向的显著图，将字符区域手工聚合成文本行，并结合预测的文本块salient map，得到精准的预测结果。

* 模型结构

![](/images/holistic1.PNG)

使用基于VGG16的FCN网络，每个stage最后一层都接有side-output(图中conv部分），stage2~5还连接了上采样层，最后将所有的feature map进行融合。side-output有三个输出分别是文本块预测，字符区域预测和连接方向预测。

之所以设置side-output是为了引入梯度更快地收敛（但是损失函数并未计入这部分损失应该没有引入梯度？），由损失函数求的梯度在side-output层只更新旁路（conv, deconv, sigmoid）参数。损失函数是所有side-output和fusion output的loss加权平均，由于实验中side-output的损失很小，所以只计fusion层的损失。

![](/images/holistic3.PNG)
![](/images/holistic4.PNG)

计算文本块损失和字符区域损失的公式如上，正样本是文本区域内的像素负样本是非文本区域的像素，由于数目差异过大，在损失函数中使用平衡系数。

* 标签处理

![](/images/holistic2.PNG)

文本块salient map：bbox内值为1，其他值为0；

字符区域salient map：为了字符之间的区分性，对字符宽高尺寸各一半的区域内进行标定，注意对预测的区域进行恢复尺寸。

连接关系：将bbox的倾斜角度作为框内字符的连接角度，其中倾斜角度定义为长边与水平方向夹角。角度范围为[-90, 90]，并将角度值归一化到[0, 1]之间。不知道对于弯曲文本是如何处理的。

* 聚合方式

本文虽然对聚合方式进行了改进避免了很多人为参数，但仍然复杂且有一些前提条件。本文使用了基于图模型的聚类。

1. 首先将字符作为顶点构建图模型，使用Delaunay三角剖分（原理？）去掉错误的连接（不在同一文本行的）。

图模型中的边定义为字符间的相似性，是角度相似性和空间相似的调和平均。角度相似性（连接角度salient map）和空间相似性（根据字符的欧氏距离）定义见论文。

角度相似距离远： 同一文本行的不同word；距离相似角度相差大：只是相隔较近的字符属于不同文本行。

2. 对图顶点进行聚类。

由于不知道聚类个数，使用下面公式进行判断，达到最大值时K是最优的。这意味着所有聚类中心尽可能成线性分布（原因？）。为了处理弯曲文本，设置相似度的阈值可以区分线性或弯曲文本，大于特定值时认为是弯曲文本不做处理。（为什么使用相似度阈值何以区分线性或弯曲？实验结论？）

![](/images/holistic5.PNG)

* Experiment

ICDAR2015

precision , recall, F: 0.7226 , 0.5869 , 0.6477

ICDAR2013:

precision , recall, F:  0.8888,  0.8022,   0.8433

* 不足与改进

对模糊和高亮图片表现不佳，作者认为通过augmentation可以改善。作者认为将笔画信息已salient map形式引入可能会提升模型表现。


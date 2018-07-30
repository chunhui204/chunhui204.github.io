* 1 introduce

目标检测中类别不平衡是很常见的问题，无论在one-stage还是two-stage一二阶段negative bbox数量都远高于positive。这无法直接计算类别loss，
很常见的是使用negative hard mining挑选出一部分negative框进行计算。在RPN中我们使用随机挑选的方式，保证参与计算loss的正负样本比1:1，在SSD中（忘了再看一看），本文同样使用hard mining来避免不平衡。

* RPN

在介绍一次RPN：我们把开始生成的基准叫anchor，RPN输出的框叫proposal。在conv5_3每个位置都有多个anchor，如果RPN输出为0那么可以认为proposal的初始值就是这些anchor。（论文把这些proposal直接叫成anchor容易引起混乱，注意理解，之后也把这部分叫anchor）。
根据proposal与gt的IOU将其分为正负样本，由于负样本数目多计算loss时有不平衡问题，所以做了抽样各取128个送入loss。那么问题来了，每次迭代时都使用128个positive anchor计算坐标误差产生梯度，不妨有离gt不太远的负样本根据不属于他们的梯度误打误撞拟合到gt附近，但是对于离得较远，与任何gt都没交集的proposal
他们永远不会拟合到gt附近。对于这部分proposal参数更新就是浪费计算资源了。如果滤除negative程度不高的proposal，他们之后可能会拟合到gt位置，滤除就会降低recall，
所以能做的是滤除negative程度很高的，本文就是这么做的。


* 2.网络结构

refineDet就是用FPN对SSD进行改进。SSD是直接在原始网络bottom-up部分做预测回归，而引入FPN后分类器和回归器应该和up-bottom部分连接。除此之外
RefineNet在bottom-up部分也没闲着，他在这一部分refine anchor主要作用还是回归了anchor的坐标，形成两步回归的结构。作者认为对one-stage方法都只对anchor回归一次，
很难将anchor拟合到精确位置。作者吧bottom-up部分叫refine部分用来微调anchor，up-bottom部分叫做目标检测部分，用的是refine后的anchor。

![](/images/refine1.PNG)

RefineNet生成anchor的方式和FPN结合RPN是一样的，在每一层配置单一尺度的anchor，不同层间anchor尺度不相同。在每一层上anchor比例都是1:1,1:2,2:1.
图中上面的部分就是FPN的bottom-up，在这一部分做了对negative anchor的滤除和对anchor分类回归。

滤除negative anchor：对于negative score大于阈值(0.99)的anchor直接滤掉，不再参与梯度更新，不会送入目标检测部分。但是滤除之后负样本数量还是远多于正样本的，还是用了hard mining才计算loss。
选取标准没看懂？在目标检测部分进行同样进行hard mining。

图中的迁移连接部分就是FPN的lateral connection。不过他的网络结构比FPN复杂。

![](/images/refine2.PNG)

* 3.Experiment

 是唯一一个one-stage方法中AR过80的。虽然使用了FPN但输入分辨率仍然对性能影响很大，可见Refine320还是比Refine512性能要差的。Refine320+是指在测试阶段是用了多尺度测试。
 
 * 4 亮点
 
 1. FPN结合SSD对小目标检测效果改善应该很大，而且SSD在语义等级相差较大的特征图上做预测效果不好。
 2. 对anchor进行两步回归，由于在计算loss时还是得用hard mining，滤除anchor对精度影响并不大。

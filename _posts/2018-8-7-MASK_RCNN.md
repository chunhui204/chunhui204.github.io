* network

![](/images/mrcnn1.PNG)

mask rcnn与faster rcnn的第一点不同是：使用了更强大的骨架网络resnet101+fpn，上图左边是resnet50为骨架网络的结构，右边是resnet101+fpn为骨架网络的结构。

第二点是：ROI Align
ROI Pool做的事是将原图上的ROI映射到最后一层的特征图上，由于骨架网络的步长为16，所以ROI的尺寸会变为[1/16]，这是一个取整操作。然后将ROI划分网格并对网格边长进行取整，
比如说对20x20的ROI划分为7x7网格，那么边长会是2.最后对每个网格内所有像素点进行一个max pool。那么这两部取整都会造成极大的位置误差。如在第一步取整中即使约掉小于一的数，乘以16在原图中也会将ROI位置平移一大块。
由于分类任务对这样的平移是鲁棒的，但是对于mask 预测就会损失很大精度。

这里作者换成ROI Align，在映射到特征图时直接1/16,保留ROI的浮点坐标不进行取整，在划分网格时不取整边长而是直接在网格内取四个点，并用双线性插值计算这四个位置对应的值。对这四个点的值进行max/avg来得到这个网格的值。

![](/images/mrcnn2.PNG)
![](/images/mrcnn3.PNG)

第三点：增加mask 分支。这是一个全卷积网络，生成一个MxMxK的特征图，M是宽高，K是类别个数，mask预测是class special的，经过消除试验发现比class agnostic（只生成一层map）会提升一点点。
计算loss = loss_coord + loss_cls + loss_mask。在计算loss时保证ROI中pos:neg = 1:3（根据IOU），只有postive ROI参与计算坐标误差，计算mask误差时，取ROI在gt中的类别对应的mask层参与计算。

* 总结

Mask Rcnn是既能做检测又能作分割的网络，这两种任务相辅相成。它在Fast rcnn上增加了预测ROI mask的分割网络，并对ROI pool换成ROI align层。创新点有三个：1.将任务细化使得网络更容易优化，相较于传统分割网络直接预测pixel-wise class的map，
mask rcnn将分类和mask预测分开。2.计算的mask是class special的，但这比class agnostic方式提升很小，注意faster rcnn的坐标预测是class agnostic方式。3.ROI align层的使用。

单线性插值：

对两点之间点用连线上的值进行插值，$\frac{y-y_0}{x-x_0}=\frac{y_1-y_0}{x_1-x_0}$导出y的公式就是所要插的值。

双线性插值

先在x方向进行两次线性插值得到$R_1,R_2$，在这两个点之间的y方向上进行插值。

![](/images/mrcnn4.PNG)

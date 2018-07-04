---
layout: post
title: "Arbitrary-Oriented Scene Text Detection via Rotation Proposal"
date: 2018-7-4
tag: Faster-rcnn  rotational anchor
---   

* bounding  box表示

1. 使用$(x,y,w,h,\theta)$ 四点坐标方式表示bbox，其中$(x,y)$ 为bbox中心坐标，不是左上角坐标，因此不用考虑文本方向降低了标注难度；

2. $w,h$ 分别为长边，短边；（旋转bbox这样定义，对水平bbox水平方向为w垂直方向h）

3. 角度范围只要覆盖180度即可，关键在于如何取角度范围，本文${\theta}\in [\frac{-\pi}{4}, \frac{3\pi}{4}]$, 为什么这样取值？

4. 四点表示的优点：对于旋转图像做augmentation是更容易处理bbox；相较于八点表示，自由度少更容易优化。

* 网络结构

![](\images\RRPN1.PNG)

基于faster rcnn结构，使用inception-RPN并根据旋转anchor提取一系列的旋转proposal，经过可以crop旋转proposal的RROI pooling层形成固定长度的向量特征，送入分类器分为text、background，清除非文本区域。

1. anchor标定及采样

	同faster rcnn，anchor作为proposal的先验，对anchor进行标定及采样的目的是为了rpn loss服务的。本文标定anchor，postive：(1)每个GT对应IOU最大的anchor为postive；（2）anchor与GT的IOU>0.7且角度差不大于$\frac{\pi}{12}$ . negative：（1）anchor与GT的IOU<0.3；（2）anchor与GT的IOU>0.7且角度差大于$\frac{\pi}{12}$ 。对旋转anchor的标定可以参考这里。

2. anchor生成

	适应文本形状，respect ratio改为1:2, 1:5, 1:8（由于width是短边所以都是小数）；scale仍是8,16,32；角度为$\frac{-\pi}{6}，0，\frac{\pi}{6}，\frac{\pi}{3}，\frac{\pi}{2}，\frac{2\pi}{3}$ ，每个像素点对应54个anchor，所以较于faster rcnn处理时间也变为2倍。这里角度的设置恰好使anchor均匀分布在角度范围内，尽量增加正样本个数，这样就能保证更多的anchor被正确回归而提高recall。

3. 损失函数的处理

$(x,y,w,h)$处理同frcnn $,\theta$ 是直接用GT角度与anchor角度差值作为角度offset，这里不用加一层正弦函数归一化？

* 性能提升的trick

1. 使用上下文信息

是指使用文本的周围信息，yolov1网络直接从整张图像上到bbox而frcnn是在得到proposal后只crop了文本区域，周围的信息没能用到因此没用到上下文（是否这样理解？）。作者直接将GT长宽扩大1.X倍对预测结果再除回来，结果明显提升。

![](\images\RRPN2.PNG)

2.增大数据集

作者rotation图像，并将模型在MSRA-TD500， ICDAR2013，ICDAR2015多个数据集上联合训练明显提高了效果。

3. scale jitter
为了处理小目标，作者随机将图像长边scale到固定值。更多的检测到小目标能提高recall。

* Experiment

![](\images\RRPN3.PNG)

* 补充
* 倾斜IOU计算

将两个bbox的相交点，以及其中一bbox在另一bbox内的顶点计入集合P中，对P中点按时钟顺序排序，以其中一个店为准向其他点连线，将相交区域划分成多个三角形，计算三角型面积和即为相交区域面积，则IOU可求。

![](\images\RRPN4.PNG)

* 倾斜RROI pooling

将proposal和feature map一起进行仿射变换后，在按照原来的ROI pooling操作。

![](\images\RRPN5.PNG)

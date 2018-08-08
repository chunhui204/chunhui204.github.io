# 疑问

1. ROI pooling只crop proposal对应的原图区域, 是否缺少上下文信息？如果crop 1.x倍的区域是否更有利于微调bbox？

2. 在proposal_target_layer中negative样本的选择是0.1<IOU<0.5，为什么要有>0.1? 以及在0.5附近的样本类别是否变化太快，对网络有迷惑性？

>0.1的目的是选取hard negative example,这部分样本更具迷惑性更难训练，用这些样本训练模型更好一些。这算是hard negative mining的方法，但是OHEM认为这种方法不是最优的，并去掉了0.1这个限制，认为他仅凭IOU却忽略掉了一些不常见的，背景复杂的ROI。

1. 维度

第一个括号里面的每一项是第一维的元素，再进一层括号中的项是第二维的元素，最后一维的元素是最后一层括号中单个的数字。后面的维度作为前面维度的项，
当前面维度的元素修改或复制后，后面维度的元素也是跟着变得。
```
a = np.arange(2*4*4)
b = a.reshape(1,4,4,2)           #应该这样按反序来理解：最后一个2是一个只有2个元素的向量，最后的4,2代表4×2的矩阵，最后的4×4×2代表立体张量，第一个1代表只有一个这样的张量（即该张量在第四维度只有一个元素）。
c = a.reshape(2,4,4,1)        #应该这样按反序来理解：最后一个1是只有一个1个元素的向量，最后的4,1代表4×1的矩阵(可降维成一个向量)，最后的4×4×1代表立体张量(可降维成一个矩阵)，第一个2代表存在2个这样的张量(或矩阵) （即该张量在第四维度有2个元素）。
```
tf.reduce_*系列：对某个维度求和或求平均等，这个维度被消去。注意batch_norm中指定的axis是不对该轴normalize（减均值除std）。
如果对图像乘以权重，保证对每个像素点的权重都相同在通道维度上权重不同即可，权重形状是(1,1,1,C)

2. broadcast

当两张量的秩相匹配其他维度相同，只有一个维度不同而且其中一个张量维度为1时，那么相应张量会在1这个维度上tile；
当张量比另一个张量秩少时，会以已经存在的维度为最后一个维度，然后缺失的维度补1，继续按上面的规则执行。如(2,1)和（2,）==>(2,1),(1,2) ==>(2,2),(2,2)

```
tf.variable_scope(scope, reuse=tf.AUTO_REUSE)
会在创建的变量和op节点，其名前增加前缀scope，对于重用的变量他的名字前缀会是scope_1，但他们是共享参数的，通过tf.trainable_variables()查看可知他们是同一个变量
tf.name_scope只修饰操作名称，为tf.conv等封装函数指定name，是指定的variable_name即修饰变量叶修饰操作。变量和操作命名本身是定义作用域所以尽量指定name。如下
with tf.variable_scope("CONV", reuse=tf.AUTO_REUSE):
    with tf.name_scope("tower_{}".format(1)):
        w = tf.get_variable('w', (10, 224, 224, 3), tf.float32)
        z = tf.layers.conv2d(w, filters=3, kernel_size=1, strides=1,
                             padding="same", name='conv_same')
变量w只有名称CONV/w, 卷积部分有变量名称CONV/conv_same/bias, CONV/conv_same/kernel,操作名称 CONV/tower_1/conv_same/BiasAdd,
CONV/tower_1/conv_same/Conv2D, （z是一个操作节点，返回的是卷积的最后一个操作BiasAdd），具体可用tfdbg查看。
```
3. control dependency

如果你的计算图中只包含Tensor（中间量），那么不用担心控制依赖问题，在计算tensor的时候库会为我们决定出一组依赖关系的op。如果我们进行的操作中含有变量，由于变量
可以由很多运算更新，我们要得到的操作希望在变量更新后进行，但是变量更新运算和我们定义的运算是不具有控制依赖关系的，如果不强制他们之间的运行顺序，在运行定义运算的时候是不会运行更新运算的。
```
with tf.control_dependencies([b,c]):
    assign = tf.assign(a, 5)
```
定义b,c运算一定在assign之前执行，上下文中的运算assign一定要在上下文中定义，如果是外面定义的运算只把tensor拿过来是没有效果的，注意operator=等操作都不是
节点运算，可以通过tf.identify(tensor)将tensor转为op。

NOTE： 对于tf.layers.batch_norm必须要添加到控制以来才会运行，原因如上。
```
update_ops = tf.get_collection(tf.GraphKeys.UPDATE_OPS)
  with tf.control_dependencies(update_ops):
    train_op = optimizer.minimize(loss)
```

4. python *args, **kwargs

当函数参数中含有*args, **kwargs时，表明该处应传入元组参数或字典参数的打开形式，参数个数不限。如:

```
def func(a, *args) 调用时应该func(10,1,3,2,1,4)
def func(a, **kwargs) 调用时应该func(10,'c':1, 'd': 14, 'e':10)或者func(10,**{'c':1, 'd': 14, 'e':10})
```
*作用于tuple（list），**作用于dict时表示将元组或字典拆开，变成元素散列的形式。如
```
func(*(1,2,3)) == func(1,2,3) not func((1,2,3))
func(**{'a':1,'b':2}) == func(a=1,b=2)
```

zip(a,b,c...) 注意参数是散列形式，会将a,b,c中对应位置的元素组合放到list里。如果直接向zip传入一个list会认为是一个元素，应该zip(*[])

5. tf.group()与tf.control_dependencies()

tf.group(a,b,c...)将多个op合并为一个op，注意参数是多个op，而不是op list，传入list参数要tf.group(*[e1,e2...]).相对于control_dependencies, group
合并的op是没有先后顺序的，等价于
 ```
 train_op = tf.group(a,b) ==>
 with tf.control_dependencies([a,b]):
    train_op = tf.no_op()
 ```
 /************相关API理解：**********************/
 ```
1. tf.stack([tensor, tensor...], axis)在制定轴stack，增加一个维度 ，  
   tf.concat([tensor, tensor...], axis)在制定轴连接tensor，不增加维度
   假设a=[[1,2,3],    b=[[7,8,9],                                 [[[1 2 3]      tf.concat([a,c], axis=0) [[1,2,3,7,8,9],
        [4,5,6]]       [6,7,8]]    -->  tf.stack([a,b], axis=0)   [4 5 6]]  ->                             [4,5,6,6,7,8]]
                                                                  [[7 8 9]
                                                                  [6 7 8]]]
   tf.unstack(tensor, axis) 在制定轴将tensor拆开，数量为该轴维度，维度减少一个。
   tf.split(tensor, num_or_size_splits, axis):拆开但不减少维度，num=int:均匀分；num=list:按list元素给定的形状分。
                    tf.split(tensor(2,11), [1,5,5])
```
/***********************************/

6. python篇--内置函数

这种设置用于向某函数传入回调函数，需要出规定参数以外的其他参数，这时候需要定义内置函数，外层函数传入额外参数，内置函数仍然是回调函数的格式，外层函数直接返回内层函数的函数名。
```
def _out(other_params):
    #do with other params...
    def _in(origin_params):
        #do som thing
    return _in
    
api_func(_out(params))  #因为_out(params) == _in
```
命名规范：
类成员变量用self.定义，私有成员变量和函数用前置两个下划线定义，不能被本类以外的其他作用域调用。共有的成员前不加下划线或加一个。静态函数用@staticmethod修饰，使用同c++。

7. glabal_step 获取有两种方法，第一种直接tf.train.get_global_step(),第二种定义变量
 ```
 global_step =tf.get_variable(0, dtype=tf.int32,  trainable=False)
 ```
 但是两者都要传入梯度更新函数中，使变量得到更新。
 ```
 opt.minimize(global_step=global_step) #在这两两个函数中，每当权重更新一次，global_step变量加1.
 or opt.apply_gradients(global_step=global_step) #或者将变量换成tf.train.get_global_step()
 ```
 8. 多gpu并行
 
 （1）gpu设置  
 ```
config = tf.ConfigProto()
config.gpu_options.allow_growth = True #允许内存增长模式。默认情况是预分配空间的方式即把gpu占满，防止内存碎片。如果使用allow_growth内存会一点点增加但不会减少，同时为了避免内存碎片。
config.gpu_options.per_process_gpu_memory_fraction = 0.4#限制使用每块gpu内存的比例，不常用
 ```
 （2）会设置一个参数设备（ps）和多个计算设备（worker)，在ps上放置参数在worker放置计算节点，但每个worker上也会保存模型参数的副本（tower), 这样是把参数放置在cpu而计算op定义在gpu，当然也可以图省事都定义在ps上。
 首先数据会被分成多分送入不同的worker，计算相应的loss, gradient, prediction，最后将这些量进行平均作为最终结果来更新模型。在定义变量时一定要使用变量重用的方式，这能保证不同的worker和ps之间的模型共享。

9. keras模型+tensorflow训练

https://www.tensorflow.org/guide/estimators

10. 模型定义部分

1. 注意padding选择
```
VALID方式：output_shape = (H - f + 1)/s   SAME方式：output_shape= [(H-f+2p)/s ] + 1
```
在tensorflow中，valid是舍弃像素的方式，卷积不到的部分直接舍弃；same补零不是在一圈补零，而是所需的补零个数2p为奇数时遵循左奇右偶的方式，和理论上的不灵方式不一样，所以在这种情况时使用tf.pad + conv(valid)方式代替系统默认的same方式。

在定义网络是卷积部分不单独使用valid（前面有pad例外），因为可以由same代替。先通过输入输出形状求出2p是奇还是偶，偶数的话same方式和理论相同，技术的话要先tf.pad再conv(valid)。池化部分同样如此，所以conv3*3, s=1 和pool2*2, s=2是不用tf.pad的直接same即可。

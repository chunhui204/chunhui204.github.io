1.

第一个括号里面的每一项是第一维的元素，再进一层括号中的项是第二维的元素，最后一维的元素是最后一层括号中单个的数字。后面的维度作为前面维度的项，
当前面维度的元素修改或复制后，后面维度的元素也是跟着变得。

2. broadcast

当两张量的秩相匹配其他维度相同，只有一个维度不同而且其中一个张量维度为1时，那么相应张量会在1这个维度上tile；
当张量比另一个张量秩少时，会以已经存在的维度为最后一个维度，然后缺失的维度补1，继续按上面的规则执行。如(2,1)和（2,）==>(2,1),(1,2) ==>(2,2),(2,2)

```
tf.variable_scope(scope, reuse=tf.AUTO_REUSE)
会在创建的变量，其名前增加前缀scope，对于重用的变量他的名字前缀会是scope_1，但他们是共享参数的，通过tf.trainable_variables()查看可知他们是同一个变量
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

当函数参数中含有*args, **kwargs时，表明该处应传入元组参数或字典参数，参数个数不限。如:

```
def func(a, *args) 调用时应该func(10,(1,3,2,1,4))
def func(a, **kwargs) 调用时应该func(10,{'c':1, 'd': 14, 'e':10})
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
7. glabal_step 获取有两种方法，第一种直接tf.train.get_global_step(),第二种
 ```
 global_step =tf.get_variable(0, dtype=tf.int32,  trainable=False)
 opt.minimize(global_step=global_step) #在这两两个函数中，每当权重更新一次，global_step变量加1.
 or opt.apply_gradients(global_step=global_step)
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

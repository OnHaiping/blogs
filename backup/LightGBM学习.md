# LightGBM

[[笔记来源于此篇文章](https://blog.csdn.net/qq_34160248/article/details/127171265)](https://blog.csdn.net/qq_34160248/article/details/127171265)

> [!note]
>
> LightGBM是一个基于树的学习算法的梯度增强框架，被设计成分布式的和高效的
>
> 优点：
>
> 1. 训练速度更快、效率更高
> 2. 降低内存使用量
> 3. 准确性更高
> 4. 支持并行、分布式和GPU学习
> 5. 能够处理大规模数据

## 原理

**lightGBM 主要基于以下方面优化，提升整体特特性：**

1. 基于Histogram（直方图）的决策树算法
2. Lightgbm 的Histogram（直方图）做差加速
3. 带深度限制的Leaf-wise的叶子生长策略
4. 直接支持类别特征
5. 直接支持高效并行

### 基于Histogram的决策树算法

直方图算法的基本特征是：

- 先将连续的浮点特征值离散化成 k 个整数，同时构造一个宽为 k 的直方图
- 在遍历数据的时候，根据离散化之后的值作为索引在直方图中累积统计量，当遍历一次数据之后，直方图累积了需要的统计量，然后根据直方图的离散值，遍历寻找最优的分割点

> [!note]
>
> 举个例子：
>
> [0 , 0.1) --> 0;
>
> [0.1 , 0.3) --> 1;

![image-20191129114925562](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/35d71fb09163c0e766d79562250dc46e.jpeg)

使用直方图算法有很多优点。首先，最明显的是 **内存消耗的降低** ， 直方图算法不仅不需要额外存储预排序的结构，而且只需要保持特征离散化之后的值，而这个值一般使用 8 位整型存储就够了，内存消耗方面可以降低为原来的 $\frac{1}{8}$

![image-20191129115042904](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/e8cbbc762feb07415ffa7e7e878eb56a.jpeg)

然后在计算方面的代价也大幅度降低

预排序算法每遍历一个特征值就需要计算一次分裂的增益，而直方图算法只需要计算k次（k可以认为是常数）

当然，Histogram不是完美的，由于特征被离散化，所以导致不是很精确，所以会对结果产生影响。
但其实上，在不同的数据集上的结果表明，离散化的分割点对最终的精度影响不是很大，甚至有时候会好一点，原因是决策树本来就是弱模型。

### LightGBM的histogram（直方图）做差加速

一个叶子节点的直方图可以由它父亲节点的直方图与它兄弟的直方图做差得到。

通常构造直方图，需要遍历这个叶子上的所有数据，但直方图做差只需要遍历直方图的k个桶

利用这个方法，LightGBM算法可以在构建一个叶子的直方图之后，可以用非常微小的代价得到它兄弟叶子的直方图，在速度上可以提升一倍

![image-20191129115130409](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/c8814e4f140ec605d604bb93dbb05d83.jpeg)

### 深度限制的leaf-wise的叶子生长策略

**level-wise** 遍历一次数据可以同时分裂同一层的叶子，容易实现多线程优化，也好控制模型复杂度，不容易过拟合

- 但是level-wise很低效的算法，这个不加区分的对别同一层的叶子，带来了跟多没必要的开销，因为实际上很多叶子的分裂增益较低，没必要进行搜索和分裂

![image-20191129114209679](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/d80f0b5be70348c5e09ed546e26860c8.jpeg)

**Leaf-wise**则是一个比较高效的策略，每次从当前所有叶子中，找到分裂增益最大的一个叶子，然后分裂，如此循环

- 因此通 level-wise相比，在分裂次数相同的情况下，Leaf-wise可以降低更多的误差，得到更好的精度
- Leaf-wise的缺点是可能会生长出比较深的决策树，产生过拟合。
- LightGBM在leaf-wise之上增加了一个最大深度的限制，在保证高效率的同时防止过拟合

![image-20191129114318484](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/7eff32ea3c1aeba06e9516b7dad4eec2.jpeg)

### 直接支持类别特征

大多数的机器学习工具都不能直接支持类别特征，一般需要把类别特征，转化到多维的0/1特征，在效率上降低了不少

而类别特征的使用是在实践中很常用的。但是基于这个考虑，lightGBM优化了对类别特征的支持，可以直接输入类别特征，不需要额外的0/1的展开。并且在决策树上增加了类别特征的决策规则

目前看来，lightGBM是第一个支持类别特征的GBDT工具

### 支持高效并行

LightGBM还具有支持高效并行的优点。LightGBM原生支持并行学习，目前支持特征并行和数据并行的两种。

- 特征并行的主要思想是在不同机器在不同的特征集合上分别寻找最优的分割点，然后在机器间同步最优的分割点。
- 数据并行则是让不同的机器先在本地构造直方图，然后进行全局的合并，最后在合并的直方图上面寻找最优分割点。

LightGBM针对这两种并行方法都做了优化:

- 在**特征并行**算法中，通过在本地保存全部数据避免对数据切分结果的通信；

![image-20191129115433096](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/22f8d736552cdbaef645e56d706c3cdc.jpeg)

数据并行中使用**分散规约**（Reduce Scatter）把直方图合并的任务分摊到不同的机器上，降低通信和计算，并利用直方图做差，进一步减少了一半的通信量

![image-20191129115550512](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/834d2145b1b996262dffb05a15303166.jpeg)

- lightGBM 演进过程
  - ![image-20191202130509524](https://picture-typora.obs.cn-north-4.myhuaweicloud.com/images/bf7ac9937b8e3272f27825bbda9a0aaf.jpeg)

- lightGBM优势
  - 基于Histogram（直方图）的决策树算法
  - Lightgbm 的Histogram（直方图）做差加速
  - 带深度限制的Leaf-wise的叶子生长策略
  - 直接支持类别特征
  - 直接支持高效并行
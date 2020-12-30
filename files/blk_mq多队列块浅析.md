# blk_mq多队列块设备浅析

## 1. 为什么要使用多队列

在主机中，多cpu运行多个线程，每个线程都能和文件系统交互，文件系统层也是用多线程和bio层交互，但是，块设备层只有一个队列：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118152449.png)

在块设备层，来自多个cpu的bio请求被放在同一个队列中，造成阻塞：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118152521.png)

因此，提出了多队列的方法，在块设备层也做成多线程：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118153007.png)

但是，在块设备层实现多个队列并不能像文件系统一样考虑，因为块设备层需要与硬件交互，这需要硬件也支持多队列，最理想的情况是，硬件支持的队列足够多，上层的每个队列（基于软件的队列），都有硬件队列和其关联。但有些时候，硬件支持的队列有限，就形成如上图的关联关系：上图中有3个硬件队列，但是，上层总共形成了6个队列（cpu到文件系统到bio层，都是6个基于软件的队列），因此，到达块设备层时，块设备会将2个基于软件的队列和1个硬件队列关联起来。

以下是细节图：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118153621.png)



## 2. 更多信息

在编程中，一般用到的变量名和blk_mq 中各部分的对应关系：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118161046.png)

*图源：[https://img-blog.csdnimg.cn/20191110221335681.png](https://img-blog.csdnimg.cn/20191110221335681.png)*

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118161337.png)

*图源：[https://img-blog.csdnimg.cn/20191110221243484.png](https://img-blog.csdnimg.cn/20191110221243484.png)*

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201122224336.png)

图源：[https://img-blog.csdnimg.cn/2019111022505564.png](https://img-blog.csdnimg.cn/2019111022505564.png)



blk_mq 中的 io 流：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118153824.png)

块设备层的io流：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118153959.png)

因为来自上层的多个队列在块设备层被放在不同的队列中，一个需要解决的问题是：当块设备的层提交给下层设备的bio请求完成后，如何从返回的请求（complete IO）中区分其来自块设备中的哪一个队列。一种方法是使用IPI（处理器间中断处理），示意图如下：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118154103.png)

> IPI(inter-processorinterrupt)是一种特别的中断。在对称多处理器 (SMP）环境下，它可以被任意一个处理器用来对另一个处理器产生中断。IPIs典型地被用来实现高速缓存间的一致性同步（Cache Coherency Synchronization）
>
> [https://blog.csdn.net/xkjcf/article/details/7772849](https://blog.csdn.net/xkjcf/article/details/7772849)

另一个方法是使用一串数字来标记请求来自哪个队列，即：硬件从请求中获得 tag，请求完成后，在返回给 completion IO 池的的请求中也带上该 tag，示意图如下：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118160150.png)



## 3. 使用blk_mq的效果

测试 IOPS 的结果：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201122224523.png)

可以看到，随着线程的增加，使用 blk_mq 的效果明显比没使用blk_mq的单队列好。

即使在只有一个硬件队列的情况下，增加块设备层的队列也会提升性能：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201122224600.png)

因为，当块设备层只有一个队列的时候，大部分时间都花在获取设备锁上面。



相关资料：

[Linux NVMe Driver学习笔记大合集](https://mp.weixin.qq.com/s/QdyJGwcwZ4a1tUivvhYnew)： 从NVMe驱动代码进行理解，有 blk_mq 的部分；

[Multi-Queu on block layer in Linux Kernel](https://hyunyoung2.github.io/2016/09/14/Multi_Queue/)： 从代码理解 linux 内核中的 blk_mq 的实现。



> 参考资料：http://events.static.linuxfound.org/sites/events/files/slides/vault-2016.pdf


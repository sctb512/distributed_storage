# infiniswap用到的技术

infiniswap来自 [NSDI'17](https://www.usenix.org/conference/nsdi17/technical-sessions/presentation/gu)，其代码主要用到以下技术：

1. configfs（主要）

   - [configfs-用户空间控制的内核对象配置](https://www.cnblogs.com/sctb/p/13901054.html)
   - [https://www.kernel.org/doc/Documentation/filesystems/configfs/configfs.txt](https://www.kernel.org/doc/Documentation/filesystems/configfs/configfs.txt)
   - [configfs_sample.c 理解](https://www.cnblogs.com/sctb/p/13923011.html)

2. RDMA

   - [深入浅出全面解析RDMA](https://tjcug.github.io/blog/2018/06/04/深入浅出全面解析RDMA/) 
   - [RDMA技术详解（一）：RDMA概述](https://zhuanlan.zhihu.com/p/55142557)
   - [RDMA技术详解（二）：RDMA Send Receive操作](https://zhuanlan.zhihu.com/p/55142547)
   - [RDMA编程：事件通知机制](https://www.jianshu.com/p/4d71f1c8e77c)
   - [RDMA read and write with IB verbs](https://thegeekinthecorner.wordpress.com/2010/09/28/rdma-read-and-write-with-ib-verbs/)
   - [RDMA Aware Networks Programming User Manual](https://www.mellanox.com/sites/default/files/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf)
   - https://github.com/tarickb/the-geek-in-the-corner
   - [infiniband网卡安装、使用总结](https://www.cnblogs.com/sctb/p/13179542.html)
   - [qemu-kvm安装and配置桥接和SR-IOV](https://www.cnblogs.com/sctb/p/13848201.html)

3. 内核模块

   - [Linux内核模块开发（简单）](https://www.cnblogs.com/sctb/p/13816110.html)

4. bio

   - [块设备驱动、bio理解](https://www.cnblogs.com/sctb/p/14022008.html)

   - [通用块层bio详解](https://www.cnblogs.com/cobbliu/articles/11975360.html)

5. blk_mq

   - [blk_mq多队列块设备浅析](https://www.cnblogs.com/sctb/p/14022027.html)
   - [Multi-Queu on block layer in Linux Kernel](https://hyunyoung2.github.io/2016/09/14/Multi_Queue/)

6. stackbd

   - [stackbd: Stacking a block device over another block device](https://orenkishon.wordpress.com/2014/10/29/stackbd-stacking-a-block-device-over-another-block-device/)
   - [https://github.com/OrenKishon/stackbd](https://github.com/OrenKishon/stackbd)

7. 锁，信号量机制

   - [Linux驱动中的 wait_event_interruptible 与 wake_up_interruptible 深度理解](https://blog.csdn.net/yeshangzhu/article/details/78051798)
   
   - [Linux 互斥锁](https://www.cnblogs.com/fengbohello/p/7571722.html)
   
8. 其它

   - [https://github.com/SymbioticLab/Infiniswap](https://github.com/SymbioticLab/Infiniswap)
   - [infiniswap安装](https://www.cnblogs.com/sctb/p/13900982.html)
   - [深入 kernel panic 流程](https://www.cnblogs.com/linhaostudy/p/9429511.html)
   - [https://webpages.uncc.edu/~esaule/NSF-PI-CSR-2017-website/talk_slides/slide_MosharafChowdhury.pdf](https://webpages.uncc.edu/~esaule/NSF-PI-CSR-2017-website/talk_slides/slide_MosharafChowdhury.pdf)
   - [https://webpages.uncc.edu/~esaule/NSF-PI-CSR-2017-website/posters/Chowdhury2-poster.pdf](https://webpages.uncc.edu/~esaule/NSF-PI-CSR-2017-website/posters/Chowdhury2-poster.pdf)
   - [infiniswap.xmind](https://wwa.lanzous.com/idVs0jm7qed) 提取码: 6xoa

9. more
   - [http://yphu.cn/systems/](http://yphu.cn/systems/)
   - [Storage Research Group @ Tsinghua University](http://storage.cs.tsinghua.edu.cn/)


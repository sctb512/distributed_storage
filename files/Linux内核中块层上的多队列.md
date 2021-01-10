# Linux内核中块层上的多队列

如果你想知道SSD为什么使用多队列，可以看看这篇文章：[https://kernel.dk/blk-mq.pdf](https://kernel.dk/blk-mq.pdf)



## 1. 多块层

以下关于多队列层的总结来自 [The Multi-Queue Interface Article](http://ari-ava.blogspot.com/2014/07/opw-linux-block-io-layer-part-4-multi.html)，[Linux kernel git](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=a4aea5623d4a54682b6ff5c18196d7802f3e478f) 展示了如何转换为blk-mq。

blk_mq 的API实现了两级块层设计，该设计使用两组独立的请求队列。

1. **软件暂存队列**，按CPU分配；

2. **硬件调度队列**，其数量通常与blcok设备支持的实际硬件队列数量匹配。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110162436.png" style="zoom:50%;" />

如上图所示，**软件暂存队列**和**硬件调度队列**之间的映射队列数不同。

现在，我们考虑两个队列放置不同的情况。

假设这里有3种情况：

- **软件暂存队列** > **硬件调度队列**

在这种情况下，两个或多个软件暂存队列被分配到一个硬件上下文中。而在硬件上下文将从所有关联的软件队列中拉入请求的同时，进行一次调度。

- **软件暂存队列** < **硬件调度队列**

在这种情况下，**软件暂存队列**和**硬件调度队列**之间的映射是有顺序的。

- **软件暂存队列** == **硬件调度队列**

在这种情况下，这是最简单的情况，即执行直接1：1映射。



## 2. 多队列块层中的主要数据结构

### 2.1 基本结构

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110163551.png" style="zoom:50%;" />

### 2.2 blk_mq_reg（在内核4.5中）

根据 [The Multi-Queue Interface Article](http://ari-ava.blogspot.com/2014/07/opw-linux-block-io-layer-part-4-multi.html)，blk_mq_reg 结构包含了一个新的块设备向块层注册时需要的所有重要信息。

这个数据结构包括指向 blk_mq_ops 数据结构的指针，用于跟踪多队列块层与设备驱动交互的具体例程。

blk_mq_reg 结构还保存了需要初始化的硬件队列数量等。

但是，blk_mq_reg 已经不复存在。**我们需要通过 blk_mq_ops 来了解块层和块设备之间的操作**。

因此你可以在 [kernel 3.15](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=3.15#L52) 中找到以下数据结构：

```c
struct blk_mq_reg {
  struct blk_mq_ops       *ops;
  unsigned int            nr_hw_queues;
  unsigned int            queue_depth;
  unsigned int            reserved_tags;
  unsigned int            cmd_size;       /* per-request extra data */
  int                     numa_node;
  unsigned int            timeout;
  unsigned int            flags;          /* BLK_MQ_F_* */
};
```

但是，在内核4.5中，你已经无法找到。

我认为在内核4.5中，此数据结构已更改为 [**struct blk_mq_tag_set \*set**](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L67)

```c
struct blk_mq_tag_set {
  struct blk_mq_ops       *ops;
  unsigned int            nr_hw_queues;
  unsigned int            queue_depth;    /* max hw supported */
  unsigned int            reserved_tags;
  unsigned int            cmd_size;       /* per-request extra data */
  int                     numa_node;
  unsigned int            timeout;
  unsigned int            flags;          /* BLK_MQ_F_* */
  void                    *driver_data;

  struct blk_mq_tags      **tags;

  struct mutex            tag_list_lock;
  struct list_head        tag_list;
};
```

因为内核3.15中的函数 [function(struct request_queue \* blk_mq_init_queue(<font color="red">struct blk_mq_reg \* reg</font>，void \* driver_data))](http://lxr.free-electrons.com/source/block/blk-mq.c?v=3.15#L1297) ，在内核4.5中已经更改为 [function(struct request_queue \* blk_mq_init_queue(<font color="red">struct blk_mq_tag_set \* set</font>))](https://hyunyoung2.github.io/2016/09/14/Multi_Queue/lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1961)

### 2.3 blk_mq_ops 结构（在内核4.5中）

如上文中所述，此数据结构用于多队列块层与块设备层进行通信。

在此数据结构中，执行 **blk_mq_hw_ctx** 和 **blk_mq_ctx** 之间上下文映射的函数存储在 **map_queue** 字段中。

```c
struct blk_mq_ops {
  /*
   * Queue request
   */
  queue_rq_fn             *queue_rq; // this part

  /*
   * Map to specific hardware queue
   */
  map_queue_fn            *map_queue; // this part

  /*
   * Called on request timeout
   */
  timeout_fn              *timeout;

  /*
   * Called to poll for completion of a specific tag.
   */
  poll_fn                 *poll;

  softirq_done_fn         *complete;

  /*
   * Called when the block layer side of a hardware queue has been
   * set up, allowing the driver to allocate/init matching structures.
   * Ditto for exit/teardown.
   */
  init_hctx_fn            *init_hctx;
  exit_hctx_fn            *exit_hctx;

  /*
   * Called for every command allocated by the block layer to allow
   * the driver to set up driver specific data.
   *
   * Tag greater than or equal to queue_depth is for setting up
   * flush request.
   *
   * Ditto for exit/teardown.
   */
  init_request_fn         *init_request;
  exit_request_fn         *exit_request;
};
```

### 2.4 blk_mq_hw_ctx 结构（在内核4.5中）

blk_mq_hw_ctx 结构表示与 request_queue 关联的硬件上下文。

这个对应的结构是内核4.5中的 blk_mq_ctx 结构。

```c
struct blk_mq_hw_ctx {
        struct {
                spinlock_t              lock;
                struct list_head        dispatch;
        } ____cacheline_aligned_in_smp;

        unsigned long           state;          /* BLK_MQ_S_* flags */
        struct delayed_work     run_work;
        struct delayed_work     delay_work;
        cpumask_var_t           cpumask;
        int                     next_cpu;
        int                     next_cpu_batch;

        unsigned long           flags;          /* BLK_MQ_F_* flags */

        struct request_queue    *queue;
        struct blk_flush_queue  *fq;

        void                    *driver_data;

        struct blk_mq_ctxmap    ctx_map;

        unsigned int            nr_ctx;
        struct blk_mq_ctx       **ctxs;

        atomic_t                wait_index;

        struct blk_mq_tags      *tags;

        unsigned long           queued;
        unsigned long           run;
#define BLK_MQ_MAX_DISPATCH_ORDER       10
        unsigned long           dispatched[BLK_MQ_MAX_DISPATCH_ORDER];

        unsigned int            numa_node;
        unsigned int            queue_num;

        atomic_t                nr_active;

        struct blk_mq_cpu_notifier      cpu_notifier;
        struct kobject          kobj;

        unsigned long           poll_invoked;
        unsigned long           poll_success;
};
```

### 2.5 blk_mq_ctx 结构（在内核4.5中）

如上文所述，blk_mq_ctx 作为**软件暂存队列**已分配给每个CPU。

```c
struct blk_mq_ctx {
  struct {
    spinlock_t              lock;
    struct list_head        rq_list;
  }  ____cacheline_aligned_in_smp;

  unsigned int            cpu;
  unsigned int            index_hw;

  unsigned int            last_tag ____cacheline_aligned_in_smp;

  /* incremented at dispatch time */
  unsigned long           rq_dispatched[2];
  unsigned long           rq_merged;

  /* incremented at completion time */
  unsigned long           ____cacheline_aligned_in_smp rq_completed[2];

  struct request_queue    *queue;
  struct kobject          kobj;
} ____cacheline_aligned_in_smp;
```

### 2.6 request_queue 结构（在内核4.5中）

**blk_mq_hw_ctx** 和 **blk_mq_ctx** 之间的上下文映射是建立在**blk_mq_ops** 结构的 **map_queue** 字段上的。在内核4.5中，这个映射仍然是 mq_map 字段，在与块设备相关的 request_queue 数据结构中。

```c
struct request_queue {
  /*
   * Together with queue_head for cacheline sharing
   */
  struct list_head        queue_head;
  struct request          *last_merge;
  struct elevator_queue   *elevator;
  int                     nr_rqs[2];      /* # allocated [a]sync rqs */
  int                     nr_rqs_elvpriv; /* # allocated rqs w/ elvpriv */

  /*
   * If blkcg is not used, @q->root_rl serves all requests.  If blkcg
   * is used, root blkg allocates from @q->root_rl and all other
   * blkgs from their own blkg->rl.  Which one to use should be
   * determined using bio_request_list().
   */
  struct request_list     root_rl;

  request_fn_proc         *request_fn;
  make_request_fn         *make_request_fn;
  prep_rq_fn              *prep_rq_fn;
  unprep_rq_fn            *unprep_rq_fn;
  softirq_done_fn         *softirq_done_fn;
  rq_timed_out_fn         *rq_timed_out_fn;
  dma_drain_needed_fn     *dma_drain_needed;
  lld_busy_fn             *lld_busy_fn;

  struct blk_mq_ops       *mq_ops;

  unsigned int            *mq_map;

  /* sw queues */
  struct blk_mq_ctx __percpu      *queue_ctx;
  unsigned int            nr_queues;

  /* hw dispatch queues */
  struct blk_mq_hw_ctx    **queue_hw_ctx;
  unsigned int            nr_hw_queues;

  /*
   * Dispatch queue sorting
   */
  sector_t                end_sector;
  struct request          *boundary_rq;

  /*
   * Delayed queue handling
   */
  struct delayed_work     delay_work;

  struct backing_dev_info backing_dev_info;

  /*
   * The queue owner gets to use this for whatever they like.
   * ll_rw_blk doesn't touch it.
   */
  void                    *queuedata;

  /*
   * various queue flags, see QUEUE_* below
   */
  unsigned long           queue_flags;

  /*
   * ida allocated id for this queue.  Used to index queues from
   * ioctx.
   */
  int                     id;

  /*
   * queue needs bounce pages for pages above this limit
   */
  gfp_t                   bounce_gfp;

  /*
   * protects queue structures from reentrancy. ->__queue_lock should
   * _never_ be used directly, it is queue private. always use
   * ->queue_lock.
   */
  spinlock_t              __queue_lock;
  spinlock_t              *queue_lock;

  /*
   * queue kobject
   */
  struct kobject kobj;

  /*
   * mq queue kobject
   */
  struct kobject mq_kobj;

  #ifdef  CONFIG_BLK_DEV_INTEGRITY
  struct blk_integrity integrity;
  #endif  /* CONFIG_BLK_DEV_INTEGRITY */

  #ifdef CONFIG_PM
  struct device           *dev;
  int                     rpm_status;
  unsigned int            nr_pending;
  #endif

  /*
   * queue settings
   */
  unsigned long           nr_requests;    /* Max # of requests */
  unsigned int            nr_congestion_on;
  unsigned int            nr_congestion_off;
  unsigned int            nr_batching;

  unsigned int            dma_drain_size;
  void                    *dma_drain_buffer;
  unsigned int            dma_pad_mask;
  unsigned int            dma_alignment;

  struct blk_queue_tag    *queue_tags;
  struct list_head        tag_busy_list;

  unsigned int            nr_sorted;
  unsigned int            in_flight[2];
  /*
   * Number of active block driver functions for which blk_drain_queue()
   * must wait. Must be incremented around functions that unlock the
   * queue_lock internally, e.g. scsi_request_fn().
   */
  unsigned int            request_fn_active;

  unsigned int            rq_timeout;
  struct timer_list       timeout;
  struct work_struct      timeout_work;
  struct list_head        timeout_list;

  struct list_head        icq_list;
  #ifdef CONFIG_BLK_CGROUP
  DECLARE_BITMAP          (blkcg_pols, BLKCG_MAX_POLS);
  struct blkcg_gq         *root_blkg;
  struct list_head        blkg_list;
  #endif

  struct queue_limits     limits;

  /*
   * sg stuff
   */
  unsigned int            sg_timeout;
  unsigned int            sg_reserved_size;
  int                     node;
  #ifdef CONFIG_BLK_DEV_IO_TRACE
  struct blk_trace        *blk_trace;
  #endif
  /*
   * for flush operations
   */
  unsigned int            flush_flags;
  unsigned int            flush_not_queueable:1;
  struct blk_flush_queue  *fq;

  struct list_head        requeue_list;
  spinlock_t              requeue_lock;
  struct work_struct      requeue_work;

  struct mutex            sysfs_lock;

  int                     bypass_depth;
  atomic_t                mq_freeze_depth;

  #if defined(CONFIG_BLK_DEV_BSG)
  bsg_job_fn              *bsg_job_fn;
  int                     bsg_job_size;
  struct bsg_class_device bsg_dev;
  #endif

  #ifdef CONFIG_BLK_DEV_THROTTLING
  /* Throttle data */
  struct throtl_data *td;
  #endif
  struct rcu_head         rcu_head;
  wait_queue_head_t       mq_freeze_wq;
  struct percpu_ref       q_usage_counter;
  struct list_head        all_q_node;

  struct blk_mq_tag_set   *tag_set;
  struct list_head        tag_set_list;
  struct bio_set          *bio_split;

  bool                    mq_sysfs_init_done;
};
```



## 3. 队列初始化

当一个新的使用多队列 API 的设备驱动被加载时，它会创建并初始化一个新的 blk_mq_ops 结构，并将一个新的 blk_mq_reg 的相关指针设置为它的地址。

更详细的说，除了下面的结构，其他的操作现在都是严格要求的。

但是，为了在上下文分配或 I/O 请求完成时执行特定的操作，可以指定其他操作。

作为必要的数据，驱动程序必须初始化它所支持的提交队列的数量，以及它们的大小。

其他数据也是必须的，以确定驱动所支持的命令大小，以及必须暴露给块层的特定标志。

<font color="red">但是</font>，在内核4.5版本中，**struct blk_mq_tag_set** 很重要，上面的工作就是在这个 struct blk_mq_tag_set 中实现的。

### 3.1 queue_fn

必须将其设置为负责处理命令的功能，例如，通过将命令传递给低级驱动程序。

### 3.2 map_queue

执行硬件和软件上下文之间的映射。

### 3.3 blk_mq_init_queue 函数（在内核4.5中）

在为设备相关的 gendisk 和 request_queue 做好准备后，驱动调用 blk_mq_init_queue 函数。(在内核4.5中)

这个函数初始化硬件和软件上下文，并执行它们之间的映射。

这个初始化例程还设置了一个备用的 make_request 函数，代替了传统的请求提交路径，其中包括函数 blk_make_request() (在内核4.5中)的多队列提交路径(其中包括 blk_mq_make_request() 函数)。

换句话说，备用的 make_request 函数是用 blk_queue_make_reqeust() 设置的。



## 4. 提交请求

设备初始化用 blk_mq_make_request() 代替了传统的块 I/O 提交函数(在内核4.5中)，让多队列结构从上层的角度来使用。

多队列块层使用的 make_request (在内核4.5中不存在) 函数包含了从进程阻塞中获益的可能性，但只适用于支持单个硬件队列或异步请求的驱动。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110172537.png" style="zoom:50%;" />

如果请求是同步的，并且驱动主动使用多队列接口，则不会阻塞。

如果允许阻塞，make_request 函数也会执行请求合并，先在任务的阻塞列表里面搜索一个候选者。

最后在映射到当前 CPU 的软件队列中，提交路径不涉及任何 I/O 调度相关的回调。

**最后，make_request 会立即将任何同步请求发送到相关硬件队列中，而在 async 或 flush 请求的情况下，它会延迟这个过渡，以便后续的合并和更高效的调度。**



## 5. 请求调度

如果一个 I/O 请求是同步的（因此不允许在多队列块层中阻塞），它对设备驱动程序的调度是在同一请求的上下文中进行的。

如果请求是 async 或 flush，则存在任务阻塞。调度的顺序如下：

1. 在提交另一个I/O请求给与同一硬件队列相关联的软件队列时。
2. 在reqeust提交过程中安排的延迟工作被执行时。

多队列块层主要的 排队等候 函数是 blk_mq_run_hw_queue() (在内核4.5代码中)，它基本依赖于另一个由其 blk_mq_ops 结构的 queue_rq (在内核4.5中)字段指向的驱动专用例程。

<font color="red">重要</font>：我们必须检查 **blk_mq_run_hw_queue** 和 **request_qu** 之间的关系！

<font color="red">重要</font>：这个函数可以延迟队列的任何运行，同时它可以立即向驱动发送一个同步请求。



内部函数 __blk_mq_run_hw_queue() (在内核4.5中)，在 reqeust 是同步的情况下被 blk_mq_run_hw_queue() (在内核4.5代码中)调用，首先加入与当前服务的硬件队列相关联的任何软件队列，然后它将结果列表与已经在调度列表中的任何条目加入。

在收集了所有待服务的条目之后，函数 __blk_mq_run_hw_queue() (在内核4.5中)处理它们（这些条目），启动每个 reqeust，并通过它的 queue_rq 函数将其传递给驱动。

该函数最后通过重排或删除相关请求处理可能出现的错误。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110174014.png" style="zoom:50%;" />



> 原文：https://hyunyoung2.github.io/2016/09/14/Multi_Queue/


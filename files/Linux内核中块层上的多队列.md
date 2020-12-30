# Linux内核中块层上的多队列



## 多块层

此多队列层的摘要来自[《多队列接口》文章](http://ari-ava.blogspot.com/2014/07/opw-linux-block-io-layer-part-4-multi.html)

[Linux内核git](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=a4aea5623d4a54682b6ff5c18196d7802f3e478f)向您展示了如何转换为blk-mq

blk_mq API实现了两级块层设计，该设计使用两组独立的请求队列。

**软件登台队列，按CPU分配**

**硬件调度队列，其数量通常与blcok设备支持的实际wareware队列的数量匹配。**

![img](https://hyunyoung2.github.io/img/Image/SSD-Solid_State_Drives/2016-09-14-Multi_Queue/mutil_queue_basic.png)

如您所见，上面的图片。

**软件登台队列**和**硬件调度队列**之间的映射队列数不同

现在，让我们考虑两个队列放置不同的情况。

假设这里有3种情况。

1. **软件登台队列**>**硬件调度队列**

在这种情况下，将两个或多个**软件登台队列**分配给硬件上下文之一。在硬件上下文将从所有关联的软件队列中拉入请求的同时执行调度。

1. **软件登台队列**<**硬件调度队列**

在这种情况下，**软件登台队列**和**硬件调度队列**之间的映射是顺序的。

1. **软件登台队列**==**硬件调度队列**

在这种情况下，这是最简单的情况。即执行直接1：1映射。

# 多队列块层中的主要数据结构。

## 基本建筑

![img](https://hyunyoung2.github.io/img/Image/SSD-Solid_State_Drives/2016-09-14-Multi_Queue/multi-queue2-block%20layer.png)

## 1. blk_mq_reg（在内核4.5中）

根据[《多队列接口文章》](http://ari-ava.blogspot.com/2014/07/opw-linux-block-io-layer-part-4-multi.html)，

blk_mq_reg结构包含在将新块设备注册到块层期间的所有重要信息。

该数据结构包括指向blk_mq_ops数据结构的指针，该数据结构用于跟踪多队列块层将用于与设备驱动程序进行交互的特定例程。

blk_mq_reg结构还保留要初始化的硬件队列的数量，依此类推。

**但是，blk_mq_reg消失了。

**我想我需要考虑blk_mq_ops来了解块层和块设备之间的操作**

因此，您可以修改[内核3.15中](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=3.15#L52)的数据结构

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

但是，在内核4.5中，您无法检查此数据结构。

我认为此数据结构已更改为[**struct 4.5中的struct blk_mq_tag_set \* set**](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L67)

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

因为在内核3.15中，[**function（struct request_queue \* blk_mq_init_queue（struct blk_mq_reg \* reg，void \* driver_data））**](http://lxr.free-electrons.com/source/block/blk-mq.c?v=3.15#L1297)更改为[**function（struct request_queue \* blk_mq_init_queue（struct blk_mq_tag_set \* set））**](https://hyunyoung2.github.io/2016/09/14/Multi_Queue/lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1961)

## 2. [blk_mq_ops结构（在内核4.5中）](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L106)

如您在上文中所读到的，此数据结构用于将多队列块层与块设备的层进行通信。

在此数据结构中，执行**blk_mq_hw_ctx**和**blk_mq_ctx**之间的映射上下文的函数存储在**map_queue**字段中。

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

## 3. [blk_mq_hw_ctx结构（在内核4.5中）](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L21)

blk_mq_hw_ctx结构表示与request_queue关联的硬件上下文。

这个对应的结构是[blk_mq_ctx结构。（在内核4.5中）](http://lxr.free-electrons.com/source/block/blk-mq.h?v=4.5#L6)

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

## 4. [blk_mq_ctx结构。（在内核4.5中）](http://lxr.free-electrons.com/source/block/blk-mq.h?v=4.5#L6)

如您在上文中所见，blk_mq_ctx作为**软件登台队列**已分配给每个CPU。

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

## 5. [request_queue结构（在内核4.5中）](http://lxr.free-electrons.com/source/include/linux/blkdev.h?v=4.5#L283)

**blk_mq_hw_ctx**和**blk_mq_ctx**之间的上下文映射是建立在**blk_mq_ops结构的****map_queue字段上**的。映射保留为与内核4.5中的块设备相关的[**request_queue**](http://lxr.free-electrons.com/source/include/linux/blkdev.h?v=4.5#L283)数据结构的[**mq_map**](http://lxr.free-electrons.com/source/include/linux/blkdev.h?v=4.5#L312)

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

## 队列初始化

加载使用多队列API的新devie驱动程序时，它将创建并初始化新的blk_mq_ops结构，并将新blk_mq_reg的关联指针设置为其地址。

更详细地讲，除了下面的结构，现在严格要求其他操作，

但是，可以指定其他操作以便在分配上下文或完成I / O请求时执行特定操作。

根据必要的数据，驱动程序必须初始化其支持的提交队列的数量及其大小。

需要其他数据来确定驱动程序支持的命令的大小以及必须公开给块层的特定标志。

**但是，在内核版本4.5中，[struct blk_mq_tag_set](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L67)很重要。以上工作是在struct blk_mq_tag_set中实现的。**

### queue_fn

必须将其设置为负责处理命令的功能，例如，通过将命令传递给低级驱动程序。

### map_queue

执行硬件和软件上下文之间的映射。

### [blk_mq_init_queue函数（在内核4.5源中）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1961)

准备好与设备相关的gendisk和request_queue之后。它（驱动程序）调用[blk_mq_init_queue函数（在内核4.5源中）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1961)

此功能初始化硬件和软件上下文并执行它们之间的映射。

该初始化例程还设置了一个替代的make_request函数，以替代常规的请求提交路径，该路径将包括功能[blk_make_request（）（在内核4.5代码中）](http://lxr.free-electrons.com/source/block/blk-core.c?v=4.5#L1342)多队列提交路径（而包括[blk_mq_make_request（）（在内核4.5中）代码）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1243)）。

换句话说，备用make_request函数是使用blk_queue_make_reqeust（）帮助器设置的。

## 提交请求

设备初始化用[blk_mq_make_request（）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L1243)（在内核4.5代码中）代替了常规的块I / O提交功能，从而使多队列结构成为上层的视角。

mutil-queue块层使用的make_request函数（在内核4.5中不存在）可以从每个进程的插入中受益，但仅适用于支持单个硬件队列的驱动程序或异步请求。

![img](https://hyunyoung2.github.io/img/Image/SSD-Solid_State_Drives/2016-09-14-Multi_Queue/process_state.png)

如果请求是同步的，并且驱动程序主动使用多队列接口，则不会执行任何插入。

如果允许插入，make_request函数还会执行请求合并，首先在任务的插入列表中搜索候选对象，

最后，在映射到当前CPU的软件队列中，提交路径不涉及任何与I / O调度相关的回调。

**最后，make_request将任何同步请求立即发送到相应的硬件队列，同时它会延迟异步或刷新请求的转换，以允许后续合并和更有效的调度。**

## 要求派遣

如果一个I / O请求是同步的（因此不允许它形成多队列块层的插入）

在相同请求的上下文中执行上下文时，将其分派到设备驱动程序。

如果请求是异步或刷新，则说明存在任务插入，

调度按以下时间执行：

1-在将另一个I / O需求提交给与同一硬件队列关联的软件队列的情况下

2-执行要求提交期间安排的延迟工作。

多队列块层的主要队列运行功能是[blk_mq_run_hw_queue（）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L861)（在内核4.5代码中），它基本上依赖于其blk_mq_ops结构的[queue_rq](http://lxr.free-electrons.com/source/include/linux/blk-mq.h?v=4.5#L106)（在内核4.5中）字段所指向的另一个特定于驱动程序的例程。

!!我必须检查**blk_mq_run_hw_queue和request_qu**之间的关系**！**

!!此功能延迟了异步请求的任何队列运行，同时它立即将同步请求分派给驱动程序。

内部函数[__blk_mq_run_hw_queue（）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L727)（在内核4.5代码中），如果要求同步，则由[blk_mq_run_hw_queue（）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L861)（在内核4.5代码中）调用，如果需要同步，则首先加入与当前正在使用的硬件队列关联的任何软件队列，然后再加入结果列表以及调度列表中已经存在的任何条目。

收集了所有要提供的条目后，函数[__blk_mq_run_hw_queue（）](http://lxr.free-electrons.com/source/block/blk-mq.c?v=4.5#L727)（在内核4.5代码中）处理它们（条目），启动每个请求，并将其与queue_rq函数一起传递给驱动程序。

该函数最终通过重新排队或删除关联的请求来处理可能的错误。

![img](https://hyunyoung2.github.io/img/Image/SSD-Solid_State_Drives/2016-09-14-Multi_Queue/Mutil_queue3.png)
# 块设备驱动、bio理解

别人写过的内容，我就不写了。贴一下大佬的博客，写的非常好：

1. [块设备驱动实战基础篇一 （170行代码构建一个逻辑块设备驱动）](https://blog.csdn.net/u011013137/article/details/9092711)

2. [块设备驱动实战基础篇二 （继续完善170行过滤驱动代码至200行）](https://blog.csdn.net/u011013137/article/details/9092833)
3. [块设备驱动实战基础篇三 （BIO请求回调机制）](https://blog.csdn.net/u011013137/article/details/9093045)
4. [块设备驱动实战基础篇四 （逐渐成型，加入ioctl通信机制）](https://blog.csdn.net/u011013137/article/details/9093111)

较遗憾的是，该博主的 <font color="green">块设备驱动实战高级篇</font> 自2013年后就未更新了，可能有更重要的事忙。



复制进去的demo代码，直接编译会报错，做了轻微改动。我的实验环境如下：

系统：ubuntu 16.04

内核：4.15.0-122-generic

架构：x86-64



## 1. 编译生成demo

创建头文件：

```shell
vim fbd_device.h
```

fbd_device.h 的内容如下：

```c
#ifndef  _FBD_DRIVER_H
#define  _FBD_DRIVER_H

#include <linux/init.h>
#include <linux/module.h>		/* 写内核模块都需要包含该头文件 */
#include <linux/blkdev.h>		/* 写内核块设备驱动必须要包含的三个头文件：blkdev.h, bio.h, genhd.h */
#include <linux/bio.h>
#include <linux/genhd.h>

#define SECTOR_BITS             (9)		/* 用来表示扇区的比特数，对于块设备，扇区是其最小的传输和存储单元，默认扇区大小是512字节，这里的9代表将512换算为二进制需要多少位描述，很快可以算出来：2＾9 = 512 */
#define DEV_NAME_LEN            32		/* 过滤块设备的名字最长为32个字节 */
#define DEV_SIZE                (512UL<< 20)   /* 过滤块设备大小是512M，1左移20位是1M，再乘以扇区大小即为512M */

#define DRIVER_NAME            "filter driver"	/* 给驱动程序注册的名字"fbd_driver" */

#define DEVICE1_NAME           "fbd1_dev"		/* 过滤块设备驱动程序创建的过滤块设备名字"fbd1_dev" */
#define DEVICE1_MINOR           0
#define DEVICE2_NAME           "fbd2_dev"		/* 过滤块设备驱动程序创建的过滤块设备名字"fbd2_dev" */
#define DEVICE2_MINOR           1

struct fbd_dev {		/* 结构体fbd_dev，三个成员：queue指针成员，disk指针，设备大小，该结构体描述我们创建的过滤块设备 */
  struct request_queue *queue;
  struct gendisk *disk;
  sector_t size;          /* device size in Bytes */
};

#endif
```

创建主要代码文件：

```shell
vim fbd_device.c
```

fbd_device.c 的内容如下：

```c
/**
 *  fbd-driver - filter block device driver
 *  Author: Talk@studio
 *  Modified by abin
 **/
 
#include "fbd_driver.h"

static int fbd_driver_major = 0;

static struct fbd_dev fbd_dev1 = {NULL};
static struct fbd_dev fbd_dev2 = {NULL};

static int fbddev_open(struct inode *inode, struct file *file);
static int fbddev_close(struct inode *inode, struct file *file);

static struct block_device_operations disk_fops = {
  .open = (void *)fbddev_open,
  .release = (void *)fbddev_close,
  .owner = THIS_MODULE,
};

/* 块设备被打开时调用该函数 */
static int fbddev_open(struct inode *inode, struct file *file)
{
  printk("device is opened by:[%s]\n", current->comm);
  return 0;
}

/* 块设备被关闭时调用该函数 */
static int fbddev_close(struct inode *inode, struct file *file)
{
  printk("device is closed by:[%s]\n", current->comm);
  return 0;
}

/* 仓库的加工函数，在dev_create中被调用 */
static int make_request(struct request_queue *q, struct bio *bio)		//参数1是我们的关卡请求队列，参数2是上层准备好的盒子bio请求描述结构体指针
{
  struct fbd_dev *dev = (struct fbd_dev *)q->queuedata;

  printk("device [%s] recevied [%s] io request, "
         "access on dev sector[%llu], length is [%u] sectors.\n",
         dev->disk->disk_name,
         bio_data_dir(bio) == READ ?"read" : "write",
         (long long)bio->bi_iter.bi_sector,
         bio_sectors(bio));

  bio_endio(bio);		//结束一个bio请求

  return 0;
}

/* 创建过滤设备的函数，在init函数中被调用 */
static int dev_create(struct fbd_dev *dev, char *dev_name, int major, int minor)
{
  int ret = 0;

  /* init fbd_dev */
  dev->size = DEV_SIZE;

  dev->disk = alloc_disk(1);		/* 申请仓库gendisk，返回值为gendisk结构体 */
  if (!dev->disk) {
    printk("alloc diskerror");
    ret = -ENOMEM;
    goto err_out1;
  }

  dev->queue = blk_alloc_queue(GFP_KERNEL);		/* 建立关卡，关卡申请后，可以用也可以不用，但必须申请 */

  if (!dev->queue) {
    printk("alloc queueerror");
    ret = -ENOMEM;
    goto err_out2;
  }

  /* init queue */
  blk_queue_make_request(dev->queue, (void *)make_request);		/* 仓库加工函数，即：请求处理函数make_request，第一参数是刚申请到的请求队列，第二个参数是我们写好的make_request函数名 */
  dev->queue->queuedata = dev;

  /* init gendisk */
  strncpy(dev->disk->disk_name, dev_name, DEV_NAME_LEN);	/* 给gendisk的disk_name成员赋值，也就是给仓库取名字 */
  dev->disk->major = major;		/* 把申请到的门牌号赋值给disk的成员major */
  dev->disk->first_minor = minor;	/* 赋值了一个次设备号 */
  dev->disk->fops = &disk_fops;		/* 为gendisk的文件操作函数赋值了一个函数指针集结构体 */
  set_capacity(dev->disk, (dev->size >> SECTOR_BITS));	/* 设置设备的容量大小为512M */

  /* bind queue to disk */
  dev->disk->queue =dev->queue;		/* 把申请的queue地址保存在disk中，这样仓库和关卡就绑定在一起了 */

  /* add disk to kernel */
  add_disk(dev->disk);	/*告诉内核我们的仓库需要审核一下，如果通过，那仓库就建好了 */
  return 0;

err_out2:
  put_disk(dev->disk);
err_out1:
  return ret;
}

static void dev_delete(struct fbd_dev *dev, char *name)
{
  printk("delete the device [%s]!\n", name);

  blk_cleanup_queue(dev->queue);
  del_gendisk(dev->disk);
  put_disk(dev->disk);
}

/* 内核模块入口，也是构建块设备驱动的核心部分 */
static int __init fbd_driver_init(void)
{
  int ret;

  /* register fbd driver, get the driver major number */
  fbd_driver_major =register_blkdev(fbd_driver_major, DRIVER_NAME);		/* 第一个参数是初始化的major号，第二参数是块设备驱动的名字。第一参数0时,系统会从它自己管理的情况表上查找是否有可用的号码，如果有就分配，作为regiser_blkdev的返回值 */

  if (fbd_driver_major < 0) {
    printk("get majorfail");
    ret = -EIO;
    goto err_out1;
  }

  /* create the first device */
  ret = dev_create(&fbd_dev1, DEVICE1_NAME, fbd_driver_major,DEVICE1_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE1_NAME);
    goto err_out2;
  }

  /* create the second device */
  ret = dev_create(&fbd_dev2, DEVICE2_NAME, fbd_driver_major,DEVICE2_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE2_NAME);
    goto err_out3;
  }
  return ret;

err_out3:
  dev_delete(&fbd_dev1, DEVICE1_NAME);
err_out2:
  unregister_blkdev(fbd_driver_major, DRIVER_NAME);
err_out1:
  return ret;
}

static void __exit fbd_driver_exit(void)
{
  /* delete the two devices */
  dev_delete(&fbd_dev2, DEVICE2_NAME);
  dev_delete(&fbd_dev1, DEVICE1_NAME);

  /* unregister fbd driver */
  unregister_blkdev(fbd_driver_major,DRIVER_NAME);
  printk("block device driver exit successfuly!\n");
}

module_init(fbd_driver_init);
module_exit(fbd_driver_exit);
MODULE_LICENSE("GPL");
```



创建Makefile文件：

```shell
vim Makefile
```

Makefile 文件的内容：

```makefile
obj-m := fbd_driver.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
	rm -rf Module.markers modules.order Module.symvers
```



编译块设备，生成内核模块：

```shell
make
```

> make -C /lib/modules/4.15.0-122-generic/build M=/home/abin/Desktop/share/abin_files/bio modules
>
> make[1]: Entering directory '/usr/src/linux-headers-4.15.0-122-generic'
>
>  CC [M] /home/abin/Desktop/share/abin_files/bio/fbd_driver.o
>
>  Building modules, stage 2.
>
>  MODPOST 1 modules
>
>  CC   /home/abin/Desktop/share/abin_files/bio/fbd_driver.mod.o
>
>  LD [M] /home/abin/Desktop/share/abin_files/bio/fbd_driver.ko
>
> make[1]: Leaving directory '/usr/src/linux-headers-4.15.0-122-generic'

```shell
ls -l
```

> -rw-rw-r-- 1 abin abin 4255 Nov 12 20:22 fbd_driver.c
>
> -rw-rw-r-- 1 abin abin 667 Nov 12 16:59 fbd_driver.h
>
> -rw-rw-r-- 1 abin abin 8144 Nov 12 20:30 <font color="red">fbd_driver.ko</font>
>
> -rw-rw-r-- 1 abin abin 603 Nov 12 20:30 fbd_driver.mod.c
>
> -rw-rw-r-- 1 abin abin 2584 Nov 12 20:30 fbd_driver.mod.o
>
> -rw-rw-r-- 1 abin abin 7856 Nov 12 20:30 fbd_driver.o
>
> -rw-rw-r-- 1 abin abin 229 Nov 12 20:09 Makefile
>
> -rw-rw-r-- 1 abin abin  61 Nov 12 20:30 modules.order
>
> -rw-rw-r-- 1 abin abin  0 Nov 12 20:30 Module.symvers

其中，<font color="red">fbd_driver.ko</font> 是编译好的内核模块，也就是块设备。



## 2. 运行demo

加载内核模块：

```shell
sudo insmod fbd_driver.ko
dmesg
```

> [337420.127568] **device is opened by:[systemd-udevd]**
>
> [337420.127695] **device is opened by:[systemd-udevd]**
>
> [337420.166190] **device is closed by:[systemd-udevd]**
>
> [337420.166208] **device is closed by:[systemd-udevd]**

```shell
ls -l /dev/fbd*
```

> brw-rw---- 1 root disk 252, 0 Nov 13 09:08 <font color="red">/dev/fbd1_dev</font>
>
> brw-rw---- 1 root disk 252, 1 Nov 13 09:08 <font color="red">/dev/fbd2_dev</font>

**/dev/fbd1_dev** 和 **/dev/fbd2_dev** 是创建好的过滤块设备，下面使用dd命令来使用其中一个设备。

```shell
sudo dd if=/dev/zero of=/dev/fbd1_dev bs=1M oflag=direct count=1
dmesg
```

> [337577.539057] **device is opened by:[dd]**
>
> [337577.539329] **device [fbd1_dev] recevied [write] io request, access on dev sector[0], length is [2048] sectors.**
>
> [337577.539335] **device is closed by:[dd]**

第二行是在make_request函数中输出的，第一行是fbddev_open函数中输出的，第三行是fbddev_close函数中输出的。



## 3. 关键信息

块设备驱动程序做的四件事情：

| 序号 | 函数                   | 功能                   |
| ---- | ---------------------- | ---------------------- |
| 1    | register_blk_device    | 注册并申请门牌号       |
| 2    | alloc_disk             | 申请仓库               |
| 3    | alloc_queue            | 申请仓库的关卡         |
| 4    | blk_queue_make_request | 注册仓库的加工处理函数 |

块设备核心数据结构：

| 结构体名称       | 结构体作用                   |
| ------------- | ---------------------------- |
| gendisk       | 块设备仓库                   |
| hd_struct     | 块设备分区                   |
| block_device  | 文件系统层使用的块设备描述符 |
| request_queue | 仓库的关卡（请求队列）       |
| request       | 包含多个bio的大请求          |
| bio           | 单个请求                     |

块设备核心API接口：

| API名称         | API作用          |
| --------------- | ---------------- |
| register_blkdev | 注册并申请门牌号 |
| alloc_disk             | 申请仓库                               |
| blk_alloc_queue        | 申请仓库的关卡                         |
| blk_queue_make_request | 注册仓库的加工处理函数                 |
| add_disk               | 将申请的仓库注册到内核中，成为合法仓库 |

bio关键成员：

| 类型 | 字段 | 说明 |
| ---- | ---- | ---- |
| dev_t                | bd_dev        | 块设备的主设备号和次设备号                                   |
| struct inode*        | bd_inode      | 指向bdev文件系统中块设备对应的文件索引节点的指针             |
| int                  | bd_openers    | 计数器，统计块设备已经被打开了多少次                         |
| struct mutex         | bd_mutex      | 打开或关闭的互斥量                                           |
| struct list_head     | bd_inodes     | 已打开的块设备文件的索引节点链表的首部                       |
| void*                | bd_holders    | 块设备描述符的当前所有者                                     |
| struct block_device* | bd_contains   | 如果块设备是一个分区，则指向整个磁盘的块设备描述符；否则，指向该块设备描述符 |
| unsigned             | bd_block_size | 块大小                                                       |
| struct hd_struct*    | bd_part       | 指向分区描述符的指针（如果该块设备不是一个分区，则为NULL）   |
| unsigned             | bd_part_count | 计数器，统计包含在块设备中的分区已经被打开了多少次           |
| struct gendisk*      | bd_disk       | 指向块设备中基本磁盘的gendisk结构的指针                      |
| struct list_head     | bd_list       | 用于块设备描述符链表的指针                                   |
| unsigned long        | bd_private    | 指向块设备持有者的私有数据的指针                             |

hd_struct关键成员：

| 类型 | 字段 | 说明 |
| ---- | ---- | ---- |
| sector_t | start_sect | 磁盘中分区的起始扇区               |
| sector_t | nr_sects   | 分区的长度（总共的扇区数）         |
| int      | policy     | 如果分区是只读的，则置为1；否则为0 |
| int      | partno     | 磁盘中分区的相对索引               |

gendisk关键成员：

| 类型 | 字段 | 说明 |
| ---- | ---- | ---- |
| int                                    | major        | 磁盘主设备号, 每个块设备都有唯一的主设备号，在这个块设备上建立的分区都使用这个相同的主设备号。具有相同主设备号的设备，使用相同的驱动程序。 |
| int                                    | first_minor  | 与磁盘关联的第一个次设备号。在某一个设备上首先创建的设备的初始次设备号为0，在名称中不显示，如sda；在这个设备上依次建立的其他设备，此设备号在0基础上依次加1，并在名称中显示，如sda1，sda2。 |
| int                                    | minors       | 与磁盘关联的次设备号范围。规定了可以在这个设备上创建多少个分设备（分区）。当次设备号数量是1时，表示这个设备不能被分区。 |
| char                                   | disk_name    | 磁盘的标准命名（通常是相应设备文件的规范名称）               |
| struct hd_struct                       | part0        | 磁盘的分区信息                                               |
| const struct block_device_operations * | fops         | 指向块设备操作函数集的指针                                   |
| struct request_queue *                 | queue        | 指向磁盘请求队列的指针                                       |
| void *                                 | private_data | 块设备驱动程序的私有数据                                     |
| int                                    | flags        | 描述磁盘类型的标志                                           |

块设备gendisk fops函数指针集：

| 类型 | 方法 | 参数 | 触发操作 |
| ---- | ---- | ---- | -------- |
| int  | (*open)    | struct block_device*, fmode_t                         | 打开块设备文件，增加引用计数                 |
| int  | (*release) | struct gendisk*, fmode_t                              | 关闭对块设备文件的最后一个引用，减少引用计数 |
| int  | (*ioctl)   | struct block_device*, fmode_t, unsigned,unsigned long | 在块设备文件上发出ioctl()系统调用            |

request_queue请求队列描述符中的关键字段：

| 类型             | 字段            | 说明                       |
| ---------------- | --------------- | -------------------------- |
| struct list_head | queue_head      | 待处理请求的链表           |
| make_request_fn* | make_request_fn | 设备驱动程序的请求处理函数 |

 request描述符的关键字段：

| 类型             | 字段      | 说明                                                         |
| ---------------- | --------- | ------------------------------------------------------------ |
| struct list_head | queuelist | 请求队列链表的指针                                           |
| struct bio*      | bio       | 请求中第一个没有完成传送操作的bio，不能直接对该成员进行访问；而要使用rq_for_each_bio访问 |
| struct bio*      | biotail   | 请求链表中末尾的bio                                          |

bio结构中的关键字段：

| 类型                  | 字段              | 说明                                            |
| --------------------- | ----------------- | ----------------------------------------------- |
| sector_t              | bi_sector         | 块I/O操作的第一个磁盘扇区                       |
| struct bio*           | bi_next           | 链接到请求队列中的下一个bio                     |
| struct block_device * | bi_bdev           | 指向块设备描述符的指针                          |
| unsigned long         | bi_flags          | bio的状态标志                                   |
| unsigned long         | bi_rw             | I/O操作标志                                     |
| unsigned short        | bi_vcnt           | bio的bio_vec数组中段的数目                      |
| unsigned short        | bi_idx            | bio的bio_vec数组中段的当前索引值                |
| unsigned int          | bi_phys_segments  | 合并之后bio中物理段的数目                       |
| unsigned int          | bi_size           | 需要传送的字节数                                |
| unsigned int          | bi_seg_front_size | 第一个可合并的段大小                            |
| unsigned int          | bi_seg_back_size  | 最后一个可合并的段大小                          |
| unsigned int          | bi_max_vecs       | bio的bio_vec数组中允许的最大段数                |
| struct bio_vec*       | bi_io_vec         | 指向bio的bio_vec数组中的段的指针                |
| atomic_t              | bi_cnt            | bio的引用计数                                   |
| bio_end_io_t*         | bi_end_io         | bio的I/O操作结束时调用的方法                    |
| void*                 | bi_private        | 通用块层和块设备驱动程序的I/O完成方法使用的指针 |

bio_vec结构中的字段：

| 类型         | 字段      | 说明                         |
| ------------ | --------- | ---------------------------- |
| struct page* | bv_page   | 指向段的页框中页描述符的指针 |
| unsigned int | bv_len    | 段的字节长度                 |
| unsigned int | bv_offset | 页框中中段数据的偏移量       |

核心API函数：

|函数名| 输入参数 | 返回值 | 说明  |
| --------------------- | ---------------------- | --------------------------------------------- | --------------------------------------------------- |
| int                   | register_blkdev        | unsigned int major, const char* name          | 成功返回主设备号，失败返回一个负数。                | 向系统申请注册一个名为“name”的主设备号，当主设备号设为0时由内核自动分配一个可用的主设备号；若自己指定时，需要确保不与已有设备冲突。 |      |
| struct gendisk*       | alloc_disk             | int minors                                    | 成功返回一个指向gendisk描述符的指针，失败返回NULL   | 分配一个gendisk结构，minors为次设备号的总数，一般也就是磁盘分区的数量，为1时表示该设备不能被分区，此后minors不能被修改 |      |
| struct request_queue* | blk_alloc_queue        | gfp_t gfp_mask                                | 成功返回一个指向request_queue的指针，失败时返回NULL | 申请请求队列，并给队列分配空间，该队列需要用户自己去调用blk_queue_make_request函数进行初始化，其中的make_request_fn函数也需要用户自己实现 |      |
| void                  | blk_queue_make_request | struct request_queue* q, make_request_fn* mfn | 无返回                                              | 初始化一个设备的请求队列，其参数make_request_fn需要我们自己实现 |      |
| void                  | add_disk               | struct gendisk *disk                          | 无返回                                              | 将gendisk添加到系统中。调用该函数后，会在“/dev/”下显示出块设备名字，设备名字即是disk->disk_name的内容。对add_disk()的调用必须发生在驱动程序的初始化工作完成并能响应磁盘的请求之后。 |      |
| void                  | del_gendisk            | struct gendisk* disk                          | 无返回                                              | 将gendisk从系统中删除。gendisk是一个引用计数结构，通常对del_gendisk的调用会删除gendisk中的最终计数，但是没有机制能保证其肯定发生。因此当调用此函数后，该结构可能继续存在（而且内核可能会调用我们提供的各种方法）。 |      |
| void                  | put_disk               | struct gendisk* disk                          | 无返回                                              | 释放驱动分配的gendisk结构                                    |      |
| void                  | unregister_blkdev      | unsigned int major, const char *name          | 无返回                                              | 注销驱动程序，释放申请的主设备号                             |      |
| void                  | blk_cleanup_queue      | struct request_queue *q                       | 无返回                                              | 释放分配的请求队列                                           |      |
| void                  | bio_endio              | struct bio *bio, int error                    | 无返回                                              | 返回对bio请求的处理结果，第一个参数即为被处理的bio指针，第二个参数成功时为0，失败时为-ERRNO。 |      |



## 4. 完善代码（加入bio过滤功能）

Makefile文件不变，fbd_driver.h 内容如下：

```c
#ifndef  _FBD_DRIVER_H
#define  _FBD_DRIVER_H

#include <linux/init.h>
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/bio.h>
#include <linux/genhd.h>


#define SECTOR_BITS             (9)
#define DEV_NAME_LEN            32

#define DRIVER_NAME            "filter driver"

#define DEVICE1_NAME           "fbd1_dev"
#define DEVICE1_MINOR           0
#define DEVICE2_NAME           "fbd2_dev"
#define DEVICE2_MINOR           1


struct fbd_dev {
  struct request_queue *queue;
  struct gendisk *disk;
  sector_t size;          /* devicesize in Bytes */
	
  /* new code */
  char lower_dev_name[DEV_NAME_LEN];
  struct block_device *lower_bdev;
};

#endif
```

size记录将来要创建的fbd_dev设备的容量大小，该容量需要保持与fbd_dev底层设备大小一致，lower_dev_name记录了底层设备的文件名字，lower_bdev保存着底层设备的block_device描述符。

fbd_driver.c内容如下：

```c
/**
 *  fbd-driver - filter block device driver
 *  Author: Talk@studio
 *  Modified by abin
 **/
 
#include "fbd_driver.h"

static int fbd_driver_major = 0;

/* new code */
static struct fbd_dev fbd_dev1 = {
  .queue = NULL,
  .disk = NULL,
  .lower_dev_name = "/dev/sdb",
  .lower_bdev = NULL,
  .size = 0
};
/* new code */
static struct fbd_dev fbd_dev2 = {
  .queue = NULL,
  .disk = NULL,
  .lower_dev_name = "/dev/sdc",
  .lower_bdev = NULL,
  .size = 0
};

static int fbddev_open(struct inode *inode, struct file *file);
static int fbddev_close(struct inode *inode, struct file *file);

static struct block_device_operations disk_fops = {
  .open = (void *)fbddev_open,
  .release = (void *)fbddev_close,
  .owner = THIS_MODULE,
};

/* 块设备被打开时调用该函数 */
static int fbddev_open(struct inode *inode, struct file *file)
{
  printk("device is opened by:[%s]\n", current->comm);
  return 0;
}

/* 块设备被关闭时调用该函数 */
static int fbddev_close(struct inode *inode, struct file *file)
{
  printk("device is closed by:[%s]\n", current->comm);
  return 0;
}

/* 仓库的加工函数，在dev_create中被调用 */
static int make_request(struct request_queue *q, struct bio *bio)		/* 参数1是我们的关卡请求队列，参数2是上层准备好的盒子bio请求描述结构体指针 */
{
  struct fbd_dev *dev = (struct fbd_dev *)q->queuedata;

  printk("device [%s] recevied [%s] io request, "
         "access on dev sector[%llu], length is [%u] sectors.\n",
         dev->disk->disk_name,
         bio_data_dir(bio) == READ ?"read" : "write",
         (long long)bio->bi_iter.bi_sector,
         bio_sectors(bio));

  /* new code */
  /* bio->bi_bdev = dev->lower_bdev; */
  bio_set_dev(bio, dev->lower_bdev);	/* 告诉bio请求的下一站是下层设备，即: dev->lower_bdev */
  submit_bio(bio);		/* 提交bio请求 */

  return 0;
}

/* 创建过滤设备的函数，在init函数中被调用 */
static int dev_create(struct fbd_dev *dev, char *dev_name, int major, int minor)
{
  int ret = 0;

  /* init fbd_dev */
  dev->size = DEV_SIZE;

  dev->disk = alloc_disk(1);		/* 申请仓库gendisk，返回值为gendisk结构体 */
  if (!dev->disk) {
    printk("alloc diskerror");
    ret = -ENOMEM;
    goto err_out1;
  }

  dev->queue = blk_alloc_queue(GFP_KERNEL);		/* 建立关卡，关卡申请后，可以用也可以不用，但必须申请 */

  if (!dev->queue) {
    printk("alloc queueerror");
    ret = -ENOMEM;
    goto err_out2;
  }

  /* init queue */
  blk_queue_make_request(dev->queue, (void *)make_request);		/* 仓库加工函数，即：请求处理函数make_request，第一参数是刚申请到的请求队列，第二个参数是我们写好的make_request函数名 */
  dev->queue->queuedata = dev;

  /* init gendisk */
  strncpy(dev->disk->disk_name, dev_name, DEV_NAME_LEN);	/* 给gendisk的disk_name成员赋值，也就是给仓库取名字 */
  dev->disk->major = major;		/* 把申请到的门牌号赋值给disk的成员major */
  dev->disk->first_minor = minor;	/* 赋值了一个次设备号 */
  dev->disk->fops = &disk_fops;		/* 为gendisk的文件操作函数赋值了一个函数指针集结构体 */

  /* new code */
  /* dev->lower_bdev = open_bdev_exclusive(dev->lower_dev_name, FMODE_WRITE| FMODE_READ, dev->lower_bdev); */
  blkdev_get_by_path(dev->lower_dev_name,FMODE_WRITE| FMODE_READ, dev->lower_bdev);		/* 获取底层设备的block_device数据结构指针 */
  if (IS_ERR(dev->lower_bdev)) {
    printk("Open thedevice[%s]'s lower dev [%s] failed!\n", dev_name, dev->lower_dev_name);
    ret = -ENOENT;
    goto err_out3;
  }

  dev->size = get_capacity(dev->lower_bdev->bd_disk) <<SECTOR_BITS;		/* 获取底层设备的容量大小 */
  set_capacity(dev->disk, (dev->size >> SECTOR_BITS));	/* 设置设备的容量为底层设备的容量大小 */

  /* bind queue to disk */
  dev->disk->queue =dev->queue;		/* 把申请的queue地址保存在disk中，这样仓库和关卡就绑定在一起了 */

  /* add disk to kernel */
  add_disk(dev->disk);	/*告诉内核我们的仓库需要审核一下，如果通过，那仓库就建好了 */

  return 0;

err_out3:
  blk_cleanup_queue(dev->queue);
err_out2:
  put_disk(dev->disk);
err_out1:
  return ret;
}

static void dev_delete(struct fbd_dev *dev, char *name)
{
  printk("delete the device [%s]!\n", name);

  /* new code */
  // close_bdev_excl(dev->lower_bdev);
  blkdev_put(dev->lower_bdev, FMODE_WRITE| FMODE_READ);

  blk_cleanup_queue(dev->queue);
  del_gendisk(dev->disk);
  put_disk(dev->disk);
}

/* 内核模块入口，也是构建块设备驱动的核心部分 */
static int __init fbd_driver_init(void)
{
  int ret;

  /* register fbd driver, get the driver major number */
  fbd_driver_major =register_blkdev(fbd_driver_major, DRIVER_NAME);		/* 第一个参数是初始化的major号，第二参数是块设备驱动的名字。第一参数0时,系统会从它自己管理的情况表上查找是否有可用的号码，如果有就分配，作为regiser_blkdev的返回值 */

  if (fbd_driver_major < 0) {
    printk("get majorfail");
    ret = -EIO;
    goto err_out1;
  }

  /* create the first device */
  ret = dev_create(&fbd_dev1, DEVICE1_NAME, fbd_driver_major,DEVICE1_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE1_NAME);
    goto err_out2;
  }

  /* create the second device */
  ret = dev_create(&fbd_dev2, DEVICE2_NAME, fbd_driver_major,DEVICE2_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE2_NAME);
    goto err_out3;
  }
  return ret;

err_out3:
  dev_delete(&fbd_dev1, DEVICE1_NAME);
err_out2:
  unregister_blkdev(fbd_driver_major, DRIVER_NAME);
err_out1:
  return ret;
}

static void __exit fbd_driver_exit(void)
{
  /* delete the two devices */
  dev_delete(&fbd_dev2, DEVICE2_NAME);
  dev_delete(&fbd_dev1, DEVICE1_NAME);

  /* unregister fbd driver */
  unregister_blkdev(fbd_driver_major,DRIVER_NAME);
  printk("block device driver exit successfuly!\n");
}

module_init(fbd_driver_init);
module_exit(fbd_driver_exit);
MODULE_LICENSE("GPL");
```

和上一个demo比较，查看改动的地方：

```shell
diff fbd_driver.c fbd_driver_old.c -u > fbd_driver.patch
cat fbd_driver.patch
```

补丁的内容如下：

```diff
--- fbd_driver_old.c	2020-11-16 16:51:27.344000000 +0800
+++ fbd_driver.c	2020-11-16 17:55:27.850994518 +0800
@@ -8,8 +8,22 @@
 
 static int fbd_driver_major = 0;
 
-static struct fbd_dev fbd_dev1 = {NULL};
-static struct fbd_dev fbd_dev2 = {NULL};
+/* new code */
+static struct fbd_dev fbd_dev1 = {
+  .queue = NULL,
+  .disk = NULL,
+  .lower_dev_name = "/dev/sdb",
+  .lower_bdev = NULL,
+  .size = 0
+};
+/* new code */
+static struct fbd_dev fbd_dev2 = {
+  .queue = NULL,
+  .disk = NULL,
+  .lower_dev_name = "/dev/sdc",
+  .lower_bdev = NULL,
+  .size = 0
+};
 
 static int fbddev_open(struct inode *inode, struct file *file);
 static int fbddev_close(struct inode *inode, struct file *file);
@@ -46,7 +60,10 @@
          (long long)bio->bi_iter.bi_sector,
          bio_sectors(bio));
 
-  bio_endio(bio);		//结束一个bio请求
+  /* new code */
+  // bio->bi_bdev = dev->lower_bdev;
+  bio_set_dev(bio, dev->lower_bdev);
+  submit_bio(bio);
 
   return 0;
 }
@@ -83,15 +100,29 @@
   dev->disk->major = major;		/* 把申请到的门牌号赋值给disk的成员major */
   dev->disk->first_minor = minor;	/* 赋值了一个次设备号 */
   dev->disk->fops = &disk_fops;		/* 为gendisk的文件操作函数赋值了一个函数指针集结构体 */
-  set_capacity(dev->disk, (dev->size >> SECTOR_BITS));	/* 设置设备的容量大小为512M */
+
+  /* new code */
+  // dev->lower_bdev = open_bdev_exclusive(dev->lower_dev_name, FMODE_WRITE| FMODE_READ, dev->lower_bdev);
+  blkdev_get_by_path(dev->lower_dev_name,FMODE_WRITE| FMODE_READ, dev->lower_bdev);
+  if (IS_ERR(dev->lower_bdev)) {
+    printk("Open thedevice[%s]'s lower dev [%s] failed!\n", dev_name, dev->lower_dev_name);
+    ret = -ENOENT;
+    goto err_out3;
+  }
+
+  dev->size = get_capacity(dev->lower_bdev->bd_disk) <<SECTOR_BITS;
+  set_capacity(dev->disk, (dev->size >> SECTOR_BITS));	/* 设置设备的容量 */
 
   /* bind queue to disk */
   dev->disk->queue =dev->queue;		/* 把申请的queue地址保存在disk中，这样仓库和关卡就绑定在一起了 */
 
   /* add disk to kernel */
   add_disk(dev->disk);	/*告诉内核我们的仓库需要审核一下，如果通过，那仓库就建好了 */
+
   return 0;
 
+err_out3:
+  blk_cleanup_queue(dev->queue);
 err_out2:
   put_disk(dev->disk);
 err_out1:
@@ -102,6 +133,10 @@
 {
   printk("delete the device [%s]!\n", name);
 
+  /* new code */
+  // close_bdev_excl(dev->lower_bdev);
+  blkdev_put(dev->lower_bdev, FMODE_WRITE| FMODE_READ);
+
   blk_cleanup_queue(dev->queue);
   del_gendisk(dev->disk);
   put_disk(dev->disk);
```

修改完成之后，重新编译：

```shell
make clean
make
```

先卸载内核模块，然后重新加载：

```shell
sudo rmmod fbd_driver.ko
sudo dmesg -C
sudo insmod fbd_driver.ko
dmesg
```

> [54554.965217] **device is opened by:[systemd-udevd]**
>
> [54554.965802] **device is opened by:[systemd-udevd]**
>
> [54554.998264] **device is closed by:[systemd-udevd]**
>
> [54555.018357] **device is closed by:[systemd-udevd]**

```shell
ls -l /dev/fbd*
```

> brw-rw---- 1 root disk 252, 0 Nov 13 09:08 <font color="red">/dev/fbd1_dev</font>
>
> brw-rw---- 1 root disk 252, 1 Nov 13 09:08 <font color="red">/dev/fbd2_dev</font>



## 5. 完善代码（BIO请求回调机制）

fbd_driver.h 的内容：

```c
#ifndef  _FBD_DRIVER_H
#define  _FBD_DRIVER_H

#include <linux/init.h>
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/bio.h>
#include <linux/genhd.h>


#define SECTOR_BITS             (9)
#define DEV_NAME_LEN            32

#define DRIVER_NAME            "filter driver"

#define DEVICE1_NAME           "fbd1_dev"
#define DEVICE1_MINOR           0
#define DEVICE2_NAME           "fbd2_dev"
#define DEVICE2_MINOR           1


struct fbd_dev {
        struct request_queue *queue;
        struct gendisk *disk;
        sector_t size;          /* devicesize in Bytes */

        /* new code */
        char lower_dev_name[DEV_NAME_LEN];
        struct block_device *lower_bdev;

};

/* new code for bio  call back */
struct bio_context {
        void *old_private;
        void *old_callback;
};

#endif
```

fbd_driver.c 的内容：

```c
/**
 *  fbd-driver - filter block device driver
 *  Author: Talk@studio
 *  Modified by abin
 **/
 
#include "fbd_driver.h"

static int fbd_driver_major = 0;

/* new code */
static struct fbd_dev fbd_dev1 = {
  .queue = NULL,
  .disk = NULL,
  .lower_dev_name = "/dev/loop13",
  .lower_bdev = NULL,
  .size = 0
};
/* new code */
static struct fbd_dev fbd_dev2 = {
  .queue = NULL,
  .disk = NULL,
  .lower_dev_name = "/dev/loop14",
  .lower_bdev = NULL,
  .size = 0
};

static int fbddev_open(struct inode *inode, struct file *file);
static int fbddev_close(struct inode *inode, struct file *file);

static struct block_device_operations disk_fops = {
  .open = (void *)fbddev_open,
  .release = (void *)fbddev_close,
  .owner = THIS_MODULE,
};

/* 块设备被打开时调用该函数 */
static int fbddev_open(struct inode *inode, struct file *file)
{
  printk("device is opened by:[%s]\n", current->comm);
  return 0;
}

/* 块设备被关闭时调用该函数 */
static int fbddev_close(struct inode *inode, struct file *file)
{
  printk("device is closed by:[%s]\n", current->comm);
  return 0;
}

/* new code for bio  call back */
/* bio请求回调时执行的函数 */
static int fbd_io_callback(struct bio *bio,unsigned int bytes_done, int error)
{
  struct bio_context *ctx = bio->bi_private;
  bio->bi_private = ctx->old_private;
  bio->bi_end_io = ctx->old_callback;
  kfree(ctx);

  printk("returned [%s] io request, end on sector %lu!\n", bio_data_dir(bio) == READ ?"read" : "write", bio->bi_iter.bi_sector);

  if (bio->bi_end_io) {
    int (*callback)(struct bio *bio,unsigned int bytes_done, int error) = (void *)(bio->bi_end_io);
    callback(bio, bytes_done, error);
  }

  return 0;
}

/* 仓库的加工函数，在dev_create中被调用 */
static int make_request(struct request_queue *q, struct bio *bio)		//参数1是我们的关卡请求队列，参数2是上层准备好的盒子bio请求描述结构体指针
{
  struct fbd_dev *dev = (struct fbd_dev *)q->queuedata;
  /* new code for bio  call back */
  struct bio_context *ctx;

  printk("device [%s] recevied [%s] io request, "
         "access on dev sector[%llu], length is [%u] sectors.\n",
         dev->disk->disk_name,
         bio_data_dir(bio) == READ ?"read" : "write",
         (long long)bio->bi_iter.bi_sector,
         bio_sectors(bio));

  /* new code for bio  call back */
  ctx = kmalloc(sizeof(struct bio_context), GFP_KERNEL);
  if (!ctx) {
    printk("alloc memory forbio_context failed!\n");
    bio_endio(bio);
    goto out;
  }
  memset(ctx, 0, sizeof(struct bio_context));

  ctx->old_private = bio->bi_private;
  ctx->old_callback = bio->bi_end_io;
  bio->bi_private = ctx;
  bio->bi_end_io = (void *)fbd_io_callback;

  /* new code */
  // bio->bi_bdev = dev->lower_bdev;
  bio_set_dev(bio, dev->lower_bdev);
  submit_bio(bio);

out:
  return 0;
}

/* 创建过滤设备的函数，在init函数中被调用 */
static int dev_create(struct fbd_dev *dev, char *dev_name, int major, int minor)
{
  int ret = 0;

  /* init fbd_dev */

  dev->disk = alloc_disk(1);		/* 申请仓库gendisk，返回值为gendisk结构体 */
  if (!dev->disk) {
    printk("alloc diskerror");
    ret = -ENOMEM;
    goto err_out1;
  }

  dev->queue = blk_alloc_queue(GFP_KERNEL);		/* 建立关卡，关卡申请后，可以用也可以不用，但必须申请 */

  if (!dev->queue) {
    printk("alloc queueerror");
    ret = -ENOMEM;
    goto err_out2;
  }

  /* init queue */
  blk_queue_make_request(dev->queue, (void *)make_request);		/* 仓库加工函数，即：请求处理函数make_request，第一参数是刚申请到的请求队列，第二个参数是我们写好的make_request函数名 */
  dev->queue->queuedata = dev;

  /* init gendisk */
  strncpy(dev->disk->disk_name, dev_name, DEV_NAME_LEN);	/* 给gendisk的disk_name成员赋值，也就是给仓库取名字 */
  dev->disk->major = major;		/* 把申请到的门牌号赋值给disk的成员major */
  dev->disk->first_minor = minor;	/* 赋值了一个次设备号 */
  dev->disk->fops = &disk_fops;		/* 为gendisk的文件操作函数赋值了一个函数指针集结构体 */

  /* new code */
  // dev->lower_bdev = open_bdev_exclusive(dev->lower_dev_name, FMODE_WRITE| FMODE_READ, dev->lower_bdev);
  blkdev_get_by_path(dev->lower_dev_name,FMODE_WRITE| FMODE_READ, dev->lower_bdev);
  if (IS_ERR(dev->lower_bdev)) {
    printk("Open thedevice[%s]'s lower dev [%s] failed!\n", dev_name, dev->lower_dev_name);
    ret = -ENOENT;
    goto err_out3;
  }

  dev->size = get_capacity(dev->lower_bdev->bd_disk) <<SECTOR_BITS;
  set_capacity(dev->disk, (dev->size >> SECTOR_BITS));	/* 设置设备的容量 */

  /* bind queue to disk */
  dev->disk->queue =dev->queue;		/* 把申请的queue地址保存在disk中，这样仓库和关卡就绑定在一起了 */

  /* add disk to kernel */
  add_disk(dev->disk);	/*告诉内核我们的仓库需要审核一下，如果通过，那仓库就建好了 */

  return 0;

err_out3:
  blk_cleanup_queue(dev->queue);
err_out2:
  put_disk(dev->disk);
err_out1:
  return ret;
}

static void dev_delete(struct fbd_dev *dev, char *name)
{
  printk("delete the device [%s]!\n", name);

  /* new code */
  // close_bdev_excl(dev->lower_bdev);
  blkdev_put(dev->lower_bdev, FMODE_WRITE| FMODE_READ);

  blk_cleanup_queue(dev->queue);
  del_gendisk(dev->disk);
  put_disk(dev->disk);
}

/* 内核模块入口，也是构建块设备驱动的核心部分 */
static int __init fbd_driver_init(void)
{
  int ret;

  /* register fbd driver, get the driver major number */
  fbd_driver_major =register_blkdev(fbd_driver_major, DRIVER_NAME);		/* 第一个参数是初始化的major号，第二参数是块设备驱动的名字。第一参数0时,系统会从它自己管理的情况表上查找是否有可用的号码，如果有就分配，作为regiser_blkdev的返回值 */

  if (fbd_driver_major < 0) {
    printk("get majorfail");
    ret = -EIO;
    goto err_out1;
  }

  /* create the first device */
  ret = dev_create(&fbd_dev1, DEVICE1_NAME, fbd_driver_major,DEVICE1_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE1_NAME);
    goto err_out2;
  }

  /* create the second device */
  ret = dev_create(&fbd_dev2, DEVICE2_NAME, fbd_driver_major,DEVICE2_MINOR);

  if (ret) {
    printk("create device[%s] failed!\n", DEVICE2_NAME);
    goto err_out3;
  }
  return ret;

err_out3:
  dev_delete(&fbd_dev1, DEVICE1_NAME);
err_out2:
  unregister_blkdev(fbd_driver_major, DRIVER_NAME);
err_out1:
  return ret;
}

static void __exit fbd_driver_exit(void)
{
  /* delete the two devices */
  dev_delete(&fbd_dev2, DEVICE2_NAME);
  dev_delete(&fbd_dev1, DEVICE1_NAME);

  /* unregister fbd driver */
  unregister_blkdev(fbd_driver_major,DRIVER_NAME);
  printk("block device driver exit successfuly!\n");
}

module_init(fbd_driver_init);
module_exit(fbd_driver_exit);
MODULE_LICENSE("GPL");
```



加入bio过滤功能和bio请求回调机制后的示意图如下：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201118151652.png)

*图源 [https://img-blog.csdn.net/20130614111543781](https://img-blog.csdn.net/20130614111543781)*

其中，fbd_driver 接管了来自上层（ VFS ）的bio请求，经过处理后提交给下层设备（ /dev/sdb 和 /dev/sdc ），下层设备处理完后，bio 返回也会被 fbd_driver 捕获，并进行相应处理。可以看出，fbd_driver 的作用就是在 VFS 和 底层设备之间增加了一个中间处理流程。


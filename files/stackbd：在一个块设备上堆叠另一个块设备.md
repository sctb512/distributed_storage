# stackbd：在一个块设备上堆叠另一个块设备

stackbd 是一个虚拟的块设备，它作为另一个块设备的前端，如 USB 闪存盘或循环设备。它将I/O请求传递给底层设备，同时它打印请求信息用于调试。它还有可能修改请求。
堆叠块设备（stackbd）是基于 Linux 设备映射器的代码，它是 Linux 内核中的一个块设备，RedHat 支持，用于创建逻辑卷，或者说，修改 I/O 请求的地址值和目标设备。
stackbd，暂时不修改请求。它的作用是作为一个嗅探器，对每一个请求，都会打印出它的读/写状态，块地址，页数，以及总的字节大小。
除了调试目的外，这个简单的设备是学习Linux内核中块设备编程的好方法。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110205004.png" style="zoom:50%;" />

##  1. 下载源代码并构建

首先，最好在虚拟机上工作，因为内核出错会导致操作系统崩溃，虚拟机的重启速度要快得多。

### 1.1 下载源代码

从GitHub下载源代码（或以Git或SVN检出）代码：

```shell
git clone https://github.com/OrenKishon/stackbd.git
```

### 1.2 修改错误

**实验环境如下**：

操作系统：Ubuntu 14.04

内核版本：4.4.0-148-generic

下载后，由于内核版本问题，直接编译会报错。根据错误提示直接定位报错位置：

```shell
vim ~/stackbd/module/stackbd.c +65
```

修改为如下内容：

> trace_block_bio_remap(bdev_get_queue(stackbd.bdev_raw), bio,
>             bio->bi_bdev->bd_dev, bio-><font color="red">bi_iter.</font>bi_sector);

```shell
vim ~/stackbd/module/stackbd.c +106
```

> printk("stackbd: make request %-5s block %-12llu #pages %-4hu total-size "
>
> ​      "%-10u\n", bio_data_dir(bio) == WRITE ? "write" : "read",
>
> ​     <font color="red"> (long long )</font>bio-><font color="red">bi_iter.</font>bi_sector, bio->bi_vcnt, bio-><font color="red">bi_iter.</font>bi_size);

```shell
vim ~/stackbd/module/stackbd.c +139
```

> struct block_device *bdev_raw = lookup_bdev(dev_path<font color="red">, 0</font>);

```shell
vim ~/stackbd/module/stackbd.c +173
```

> printk("stackbd: Device real capacity: %llu\n",<font color="red"> (long long)</font>stackbd.capacity);

```shell
vim ~/stackbd/module/stackbd.c +264
```

> blk_queue_make_request(stackbd.queue, <font color="red">(void *)</font>stackbd_make_request);

如果是其它内核版本，报错可能不一样，需自行修改。

### 1.3 编译

在 "module" 目录中构建内核模块：

```shell
cd ~/stackbd/module
make
```

> make -C /usr/src/linux-headers-4.4.0-148-generic SUBDIRS=/home/abin/stackbd/module modules
>
> make[1]: Entering directory `/usr/src/linux-headers-4.4.0-148-generic'
>
>  CC [M] /home/abin/stackbd/module/stackbd.o
>
>  Building modules, stage 2.
>
>  MODPOST 1 modules
>
>  CC   /home/abin/stackbd/module/stackbd.mod.o
>
>  LD [M] /home/abin/stackbd/module/stackbd.ko
>
> make[1]: Leaving directory `/usr/src/linux-headers-4.4.0-148-generic'

在 "util" 目录中构建用户端工具：

```shell
cd ~/stackbd/util
make
```

> cc  -c -o stackbd_util.o stackbd_util.c
>
> gcc -o stackbd_util stackbd_util.c



## 2. 创建用于测试的回路设备

我们需要一种设备来充当基础的“真实”设备。最简单的方法是基于文件系统中的文件创建循环设备。

创建一个100 MB的文件 *disk_file*，它将用作设备存储：

```shell
cd ~/stackbd
dd if=/dev/urandom of=disk_file bs=1024 count=100000
```

在此文件上设置循环设备 */ dev / loop0*：

```shell
sudo losetup /dev/loop0 disk_file
```

确认已创建大小为200,000（512字节块）的设备：

```shell
sudo blockdev --getsize /dev/loop0
```

> 200000

注意，循环设备在重启后不会持久化，所以一旦创建了文件disk_file，重启后只需要重复执行losetup命令。



## 3. 跟踪内核调试打印

stackbd 模块使用 printk 命令打印调试信息，所以我们需要通过跟踪 syslog 文件来跟踪它们。新开一个 shell 窗口，输入如下命令：

```shell
tail -f /var/log/syslog
```

这篇文章中的以下所有命令都应该在这个文件中产生调试信息。

##  4. 初始化堆叠设备

将 stackbd.ko 模块加载进内核，该操作只会创建新设备 */ dev / stackbd0*，而不会将其与另一个设备关联：

```shell
cd ~/stackbd/module
sudo insmod ./stackbd.ko
```

内核 syslog 输出如下：

> Jan 10 21:52:29 ubuntu kernel: [ 2873.847052] stackbd: loading out-of-tree module taints kernel.
>
> Jan 10 21:52:29 ubuntu kernel: [ 2873.847118] stackbd: module verification failed: signature and/or required key missing - tainting kernel
>
> Jan 10 21:52:29 ubuntu kernel: [ 2873.849754] stackbd: init done

使用用户端 util 使 *stackbd* 打开循环设备，它使用 *ioctl*命令来控制内核模块：

```shell
cd ~/stackbd/util
sudo stackbd_util /dev/loop0
```

> do it... </dev/loop0>
>
> OK

确认新设备 */ dev / stackbd0* 存在，并且大小与基础设备相同：

```shell
ls -l /dev/stackbd0
```

> brw-rw---- 1 root disk 251, 0 Jan 10 21:54 **/dev/stackbd0**

```shell
sudo blockdev --getsize /dev/stackbd0
```

> 200000

 执行上面命令后的 *syslog* 中的消息 应类似于：

> Jan 10 21:54:25 ubuntu kernel: [ 2990.252739] *** DO IT!!!!!!! ***
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.252739] 
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.252745] Opened /dev/loop0
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.252761] stackbd: Device real capacity: 200000
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.252763] stackbd: Max sectors: 255
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.252870] stackbd: done initializing successfully
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.254140] stackbd: make request read block 199808    #pages 1  total-size 4096    
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.254228] stackbd: make request read block 199984    #pages 1  total-size 4096    
>
> ......
>
> Jan 10 21:54:25 ubuntu kernel: [ 2990.258612] stackbd: make request read block 4096     #pages 1  total-size 4096    



## 5. 安装设备并使用

首先，在主目录下创建一个目录mnt，用于挂载。该操作只需执行一次，因为重启后目录还会保留：

```shell
mkdir ~/mnt
```

在设备上创建一个文件系统，示例为 *ext4*：

```shell
sudo mkfs.ext4 /dev/stackbd0
```

> mke2fs 1.42.9 (4-Feb-2014)
>
> Filesystem label=
>
> OS type: Linux
>
> Block size=1024 (log=0)
>
> Fragment size=1024 (log=0)
>
> Stride=0 blocks, Stripe width=0 blocks
>
> 25064 inodes, 100000 blocks
>
> 5000 blocks (5.00%) reserved for the super user
>
> First data block=1
>
> Maximum filesystem blocks=67371008
>
> 13 block groups
>
> 8192 blocks per group, 8192 fragments per group
>
> 1928 inodes per group
>
> Superblock backups stored on blocks: 
>
> ​	8193, 24577, 40961, 57345, 73729
>
> 
>
> Allocating group tables: done               
>
> Writing inode tables: done               
>
> Creating journal (4096 blocks): done
>
> Writing superblocks and filesystem accounting information: done 

在目录 *mnt*上挂载文件系统：

```shell
sudo mount -t ext4 /dev/stackbd0 ~/mnt/
```

赋予非root用户在挂载点上的读写权限：

```
sudo chmod -R 777 ~/mnt/
```

创建文件并将其写入设备中，然后，读取文件。

```
echo test > ~/mnt/1.txt
cat ~/mnt/1.txt
```



在上述操作过程中，查看详细记录I/O请求的调试打印。举个例子：

> Jan 10 22:04:33 ubuntu kernel: [ 3597.680126] stackbd: make request read  block 518          #pages 1    total-size 1024      
> Jan 10 22:04:33 ubuntu kernel: [ 3597.680270] stackbd: make request write block 16902        #pages 1    total-size 1024      
> Jan 10 22:04:38 ubuntu kernel: [ 3602.816174] stackbd: make request write block 98344        #pages 1    total-size 1024      
> Jan 10 22:04:38 ubuntu kernel: [ 3602.816194] stackbd: make request write block 98346        #pages 1    total-size 1024

##  6. 取消挂载并卸载设备

为了重新测试设备（例如在修改代码后），可以将其卸载并重新安装。

卸载文件系统（取消挂载）：

```shell
sudo umount /dev/stackbd0
```

删除模块，这将删除设备 */ dev / stackbd0*：

```shell
sudo rmmod stackbd
```

> Jan 10 22:08:00 ubuntu kernel: [ 3804.559709] stackbd: exit



## 7. 有趣的内核代码片段

在这个块设备里面打开底层的块设备，使用它的路径（在这里的例子中，路径是/dev/loop0）。用于打开块设备的函数有 lookup_dev()、bdget() 和 blkdev_get()：

```c
    struct block_device *bdev_raw = lookup_bdev(dev_path);
    printk("Opened %s\n", dev_path);
    if (IS_ERR(bdev_raw))
    {
        printk("stackbd: error opening raw device <%lu>\n", PTR_ERR(bdev_raw));
        return NULL;
    }
    if (!bdget(bdev_raw->bd_dev))
    {
        printk("stackbd: error bdget()\n");
        return NULL;
    }
    if (blkdev_get(bdev_raw, STACKBD_BDEV_MODE, &stackbd))
    {
        printk("stackbd: error blkdev_get()\n");
        bdput(bdev_raw);
        return NULL;
    }
    return bdev_raw;
```

实际上，只是将一个 I/O 请求从这个块设备重新映射到底层块设备。函数 trace_block_bio_remap() 只是简单地修改了请求的目标设备和地址，并将请求发送到另一个设备的队列中（使用 generic_make_request() 函数）：

```c
static void stackbd_io_fn(struct bio *bio)
{
    bio->bi_bdev = stackbd.bdev_raw;

    trace_block_bio_remap(bdev_get_queue(stackbd.bdev_raw), bio,
            bio->bi_bdev->bd_dev, bio->bi_sector);

    /* No need to call bio_endio() */
    generic_make_request(bio);
}
```

块设备队列函数。块设备异步处理请求（与字符设备不同）。它们定义了一个请求回调并将其注册到队列中。内核调用这个回调来处理 I/O，这个函数作为一个生产者线程，因为它只将 I/O 请求添加到一个内部列表 (struct bio list) 中，而不处理它。它向作为消费者的另一个线程发出信号，让它实际执行 I/O 操作。

```c
static void stackbd_make_request(struct request_queue *q, struct bio *bio)
{
    spin_lock_irq(&stackbd.lock);
    if (!stackbd.bdev_raw)
    {
        printk("stackbd: Request before bdev_raw is ready, aborting\n");
        goto abort;
    }
    if (!stackbd.is_active)
    {
        printk("stackbd: Device not active yet, aborting\n");
        goto abort;
    }
    bio_list_add(&stackbd.bio_list, bio);
    wake_up(&req_event);
    spin_unlock_irq(&stackbd.lock);

    return;

abort:
    spin_unlock_irq(&stackbd.lock);
    printk("<%p> Abort request\n\n", bio);
    bio_io_error(bio);
}
```

块设备 "消费者 "线程函数--等待 "生产者 "线程（也就是实际的队列线程）发出信号，表示有请求被添加到列表中，wait_event_interruptible() 是睡眠等待队列线程发出信号唤醒的函数。

```c
static int stackbd_threadfn(void *data)
{
    struct bio *bio;
    while (!kthread_should_stop())
    {
        /* wake_up() is after adding bio to list. No need for condition */ 
        wait_event_interruptible(req_event, kthread_should_stop() ||
                !bio_list_empty(&stackbd.bio_list));

        spin_lock_irq(&stackbd.lock);
        if (bio_list_empty(&stackbd.bio_list))
        {
            spin_unlock_irq(&stackbd.lock);
            continue;
        }

        bio = bio_list_pop(&stackbd.bio_list);
        spin_unlock_irq(&stackbd.lock);

        stackbd_io_fn(bio);
    }
    return 0;
}
```

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20210110221126.png" style="zoom:50%;" />



> 原文：https://orenkishon.wordpress.com/2014/10/29/stackbd-stacking-a-block-device-over-another-block-device/


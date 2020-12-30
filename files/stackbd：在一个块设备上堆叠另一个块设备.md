# stackbd：在一个块设备上堆叠另一个块设备

*stackbd*是一个虚拟块设备，充当另一个块设备（例如USB记忆棒或环路设备）的前端。它将I / O请求传递到基础设备，同时打印请求信息以进行调试。潜在地，它也可以修改请求。

堆叠式块设备（*stackbd*）基于Linux[设备映射器](https://www.sourceware.org/dm/)的代码，Linux[映射器](https://www.sourceware.org/dm/)是RedHat支持的Linux内核中的块设备，可用于创建逻辑卷或修改I / O请求的地址值和目标设备。

目前，stackbd不会修改请求。它充当嗅探器，并针对每个请求打印其读/写，块地址，页数以及总大小（以字节为单位）。

除了调试目的之外，这种简单的设备还是学习Linux内核中的块设备编程的好方法。

[![010.fig5](https://orenkishon.files.wordpress.com/2014/10/010-fig5.jpg?w=280&h=290)](https://orenkishon.files.wordpress.com/2014/10/010-fig5.jpg)

------

###  下载源代码并构建

首先，最好在虚拟机上工作，因为内核黑客攻击会导致操作系统卡住，并且虚拟机的重启速度要快得多。

从我的GitHub下载源代码（或以Git或SVN检出）代码：

https://github.com/OrenKishon/stackbd

在“模块”目录中构建内核模块，在“ util”目录中构建用户端util。
`oren@oren-VirtualBox:~/stackbd/trunk/module $ make`
`oren@oren-VirtualBox:~/stackbd/trunk/util $ make`

如果遇到编译错误，请在注释中进行描述。

------

### 创建用于测试的回路设备

我们需要一种设备来充当基础的“真实”设备。最简单的方法是基于文件系统中的文件创建循环设备。

创建一个100 MB的文件*disk_file*，它将用作设备存储：

```
oren@oren-VirtualBox:~ $ dd if=/dev/urandom of=disk_file bs=1024 count=100000
```

在此文件上设置循环设备*/ dev / loop0*：

```
oren@oren-VirtualBox:~ $ sudo losetup /dev/loop0 file_dev
```

确认已创建大小为200,000（512字节块）的设备：

`oren@oren-VirtualBox~/stackbd/trunk/module $ sudo blockdev --getsize /dev/loop0`
200000

请注意，循环设备不重新启动后仍然存在，所以一旦文件 *disk_file*被创建，只有*losetup*命令需要在重新启动后重复。

------

### 跟踪内核调试打印

该*stackbd*使用该模块打印调试消息*printk的*命令，所以我们需要通过拖尾跟随他们*的系统日志* 文件。打开第二个Shell窗口，然后移动它，以便您可以在第一个Shell窗口中与执行命令并行地看到消息。在第二个终端中：

```
oren@oren-VirtualBox:~ $ tail -f /var/log/syslog
```

这篇文章中的所有以下以下命令应在此文件中产生调试消息。

------

###  初始化并堆叠设备

将模块插入内核。这只会创建新设备*/ dev / stackbd0*，而不会将其与另一个设备关联：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo insmod ./stackbd.ko
```

使用用户端util使*stackbd*打开循环设备。它使用 *ioctl*命令来控制内核模块：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo ../util/stackbd_util /dev/loop0
```

确认新设备 */ dev / stackbd0*存在，并且大小与基础设备相同：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo blockdev --getsize /dev/stackbd0
```

200000

 上面两个命令的*syslog*中的消息 应类似于：

```
Oct 29 10:06:04 oren-VirtualBox kernel: [ 753.775151] stackbd: init doneOct 29 10:06:10 oren-VirtualBox kernel: [ 759.671404]Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671404] *** DO IT!!!!!!! ***Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671404]Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671417] Opened /dev/loop0Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671425] stackbd: Device real capacity: 200000Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671427] stackbd: Max sectors: 255Oct 29 10:06:10 oren-VirtualBox kernel: [ 759.671461] stackbd: done initializing successfully
```

------

### 安装设备并使用

首先，在主目录中为挂载创建目录*mnt*。它需要完成一次，因为它在重新启动后仍然存在：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ mkdir ~/mnt
```

在设备上创建一个文件系统，示例为*ext4*：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo mkfs.ext4 /dev/stackbd0
```

在目录*mnt*上挂载文件系统：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo mount -t ext4 /dev/stackbd0 ~/mnt/
```

为非root用户在安装点上授予读写权限：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo chmod -R 777 ~/mnt/
```

创建文件并将其写入设备中。之后，读取文件。

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ echo test > ~/mnt/1.txt
oren@oren-VirtualBox:~/stackbd/trunk/module $ cat ~/mnt/1.txt
```

测试

在上述操作过程中，查看详细说明I / O请求的调试打印。一个例子：

```
Oct 29 10:09:58 oren-VirtualBox kernel: [ 987.765196] stackbd: make request read block 544  #pages 1  total-size 1024Oct 29 10:10:00 oren-VirtualBox kernel: [ 989.880441] stackbd: make request write block 586  #pages 31 total-size 126976Oct 29 10:10:00 oren-VirtualBox kernel: [ 989.880616] stackbd: make request write block 834  #pages 29 total-size 117760Oct 29 10:10:00 oren-VirtualBox kernel: [ 989.880742] stackbd: make request write block 1064 #pages 31 total-size 126976Oct 29 10:10:00 oren-VirtualBox kernel: [ 989.880840] stackbd: make request write block 1312 #pages 30 total-size 119808
```

------

###  卸下并卸下设备

为了重新测试设备（例如在修改代码后），可以将其卸下并重新安装。

卸载文件系统：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo umount /dev/stackbd0
```

删除模块，这将删除设备*/ dev / stackbd0*：

```
oren@oren-VirtualBox:~/stackbd/trunk/module $ sudo rmmod stackbd
```

------

### 有趣的内核代码片段

使用其路径（在此示例中，路径为*/ dev / loop0*）在此块设备内部打开基础块设备 。用于打开一个块设备的功能是`lookup_dev()`，`bdget()`和`blkdev_get()`：

```
    struct block_device * bdev_raw = lookup_bdev （ dev_path ）; 
    printk （“打开的％s \ n ” ， dev_path ）; 
    如果 （ IS_ERR （ bdev_raw ））
    { 
        printk的（“ stackbd：错误开口原始设备< ％鲁> \ n ” ， PTR_ERR （ bdev_raw ））; 
        返回 NULL ; 
    }
    如果 （！ bdget （ bdev_raw - > bd_dev ））
    { 
        printk的（“ stackbd：错误bdget（）\ n ” ）; 
        返回 NULL ; 
    }
    如果 （ blkdev_get （ bdev_raw ， STACKBD_BDEV_MODE ， ＆ stackbd ））
    { 
        printk的（“ stackbd：错误blkdev_get（）\ n ” ）; 
        bdput （ bdev_raw ）;
        返回 NULL ; 
    } 
    return bdev_raw ;
```

从此块设备到基础块设备的I / O请求的实际重新映射。该函数`trace_block_bio_remap()`仅修改请求的目标设备和地址，并将请求发送到其他设备的队列（使用`generic_make_request()`）：

```
静态 无效stackbd_io_fn （结构生物*生物）
{
    生物- > bi_bdev = stackbd 。bdev_raw ; 

    trace_block_bio_remap （ bdev_get_queue （ stackbd 。 bdev_raw ），生物，
            生物- > bi_bdev - > bd_dev ，生物- > bi_sector ）; 

    / *无需调用bio_endio（）* / 
    generic_make_request（生物）; 
}
```

块设备队列功能。块设备异步处理请求（与char设备不同）。他们定义一个请求回调并将其注册到队列中。内核为I / O调用此回调。此函数充当生产者线程，因为它仅将I / O请求添加到内部列表（`struct bio list`）中，而不处理它。它向另一个充当消费者的线程发出信号，以实际执行I / O。

```
静态 空隙stackbd_make_request （结构request_queue * q ， 结构生物*生物）
{ 
    spin_lock_irq （＆ stackbd 。锁）; 
    如果 （！ stackbd 。 bdev_raw ）
    { 
        printk的（“ stackbd：请求之前bdev_raw准备，中止\ n ” ）; 
        转到 中止; 
    } 
    if  （！ stackbd 。is_active ）
    { 
        printk （“ stackbd：设备尚未激活，正在中止\ n ” ）；
        转到 中止; 
    } 
    bio_list_add （＆ stackbd 。 bio_list ，生物）; 
    WAKE_UP （＆ req_event ）; 
    spin_unlock_irq （＆ stackbd 。锁）; 

    回报; 

中止： 
    spin_unlock_irq （＆ stackbd 。锁）; 
    printk （“ < ％p >中止请求\ n \ n ” ， bio ）; 
    bio_io_error （ bio ）; 
}
```

块设备的“消费者”线程功能–等待来自“生产者”线程（是实际的队列线程）的信号，该请求已将请求添加到列表中。`wait_event_interruptible()`是睡眠等待队列线程发信号通知其唤醒的功能。

```
静态 整数stackbd_threadfn （无效 *数据）
{
    结构生物*生物; 
    while  （！ kthread_should_stop （））
    { 
        / * ake_up（）是在将bio添加到列表之后。无需条件* /  
        wait_event_interruptible （ req_event ， kthread_should_stop （） | | 
                ！ bio_list_empty （＆ stackbd 。 bio_list ））; 

        spin_lock_irq （＆ stackbd 。锁）; 
        如果 （ bio_list_empty （＆ stackbd 。 bio_list ））
        { 
            spin_unlock_irq （＆ stackbd 。锁）; 
            继续; 
        }

        生物= bio_list_pop （＆ stackbd 。 bio_list ）; 
        spin_unlock_irq （＆ stackbd 。锁）; 

        stackbd_io_fn （生物）; 
    }
    返回 0 ; 
}
```
# Linux内核模块开发（简单）

Linux系统为应用程序提供了功能强大且容易扩展的API，但在某些情况下，这还远远不够。与硬件交互或进行需要访问系统中特权信息的操作时，就需要一个内核模块。

Linux内核模块是一段编译后的二进制代码，直接插入Linux内核中，在 Ring 0（x86–64处理器中执行最低和受保护程度最低的执行环）上运行。这里的代码完全不受检查，但是运行速度很快，可以访问系统中的所有内容。

> Intel x86架构使用了4个级别来标明不同的特权级。Ring 0实际就是内核态，拥有最高权限。而一般应用程序处于Ring 3状态--用户态。在Linux中，还存在Ring 1和Ring 2两个级别，一般归属驱动程序的级别。在Windows平台没有Ring 1和Ring 2两个级别，只用Ring 0内核态和Ring 3用户态。在权限约束上，高特权等级状态可以阅读低特权等级状态的数据，例如进程上下文、代码、数据等等，但反之则不可。Ring 0最高可以读取Ring 0-3所有的内容，Ring 1可以读Ring 1-3的，Ring 2以此类推，Ring 3只能读自己的数据。
>
> 参考：https://www.cnblogs.com/KevinGeorge/p/8381149.html

## 1. 为什么要开发内核模块

编写Linux内核模块并不是因为内核太庞大而不敢修改。直接修改内核源码会导致很多问题，例如：通过更改内核，你将面临数据丢失和系统损坏的风险。内核代码没有常规Linux应用程序所拥有的安全防护机制，如果内核发生故障，将锁死整个系统。

更糟糕的是，当你修改内核并导致错误后，可能不会立即表现出来。如果模块发生错误，在其加载时就锁定系统是最好的选择，如果不锁定，当你向模块中添加更多代码时，你将会面临失控循环和内存泄漏的风险，如果不小心，它们会随着计算机继续运行而持续增长，最终，关键的存储器结构甚至缓冲区都可能被覆盖。

编写内核模块时，基本是可以丢弃传统的应用程序开发范例。除了加载和卸载模块之外，你还需要编写响应系统事件的代码（而不是按顺序模式执行的代码）。通过内核开发，你正在编写API，而不是应用程序。

你也无权访问标准库，虽然内核提供了一些函数，例如printk（可替代printf）和kmalloc（以与malloc相似的方式运行），但你在很大程度上只能使用自己的设备。此外，在卸载模块时，你需要将自己清理干净，系统不会在你的模块被卸载后进行垃圾回收。

## 2. 准备

开始编写Linux内核模块之前，我们首先要准备一些工具。最重要的是，你需要有一台Linux机器，尽管可以使用任何Linux发行版，但本文中，我使用的是Ubuntu 16.04 LTS，如果你使用的其他发行版，可能需要稍微调整安装命令。

其次，你需要一台物理机或虚拟机，我不建议你直接使用物理机编写内核模块，因为当你出错时，主机的数据可能会丢失。在编写和调试内核模块的过程中，你至少会锁定机器几次，内核崩溃时，最新的代码更改可能仍在写缓冲区中，因此，你的源文件可能会损坏，在虚拟机中进行测试可以避免这种风险。

最后，你至少需要了解一些C。对于内核来说，C++在运行时太大了，因此编写纯C代码是必不可少的。另外，对于其与硬件的交互，了解一些组件可能会有所帮助。

## 3. 安装开发环境

在Ubuntu上，我们需要运行以下代码：

```shell
sudo apt-get install build-essential linux-headers-`uname -r`
```

这将安装本文所需的基本开发工具和内核头文件。

以下示例假定你以普通用户身份而非root用户身份运行，但你具有sudo特权。sudo是加载内核模块必需的，但是我们希望尽可能在非root权限下工作。

## 4. 入门模块

让我们开始编写一些代码，准备环境：

```shell
mkdir -p 〜/src/lkm_example

cd 〜/src/lkm_example
```

启动您喜欢的编辑器（在我的例子中是vim），并创建具有以下内容的文件 lkm_example.c

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("abin");
MODULE_DESCRIPTION("A simple example Linux module.");
MODULE_VERSION("0.01");

static int __init lkm_example_init(void) {
	printk(KERN_INFO "Hello, World!\n");
	return 0;
}
static void __exit lkm_example_exit(void) {
	printk(KERN_INFO "Goodbye, World!\n");
}

module_init(lkm_example_init);
module_exit(lkm_example_exit);
```

现在，我们已经构建了最简单的内核模块，下面介绍代码的细节：

- "includes" 包括Linux内核开发所需的必需头文件。

- 根据模块的许可证，可以将MODULE_LICENSE设置为各种值。要查看完整列表，请运行：

  ```shell
  grep "MODULE_LICENSE" -B 27 /usr/src/linux-headers-`uname -r`/include/linux/module.h
  ```

- 我们将init（加载）和exit（卸载）函数都定义为静态并返回int。

- 注意使用printk而不是printf，另外，printk与printf共享的参数也不相同。例如，KERN_INFO 是一个标志，用于声明应为该行设置的日志记录优先级，并且<font color="red">不带逗号</font>。内核在printk函数中对此进行分类以节省堆栈内存。

- 在文件末尾，我们调用module_init和module_exit函数告诉内核哪些函数是内核模块的加载和卸载函数。这使我们可以任意命名这两个函数。

目前，还无法编译此文件，我们需要一个Makefile，请注意，make对于空格和制表符敏感，因此请确保在适当的地方使用<font color="red">制表符</font>而不是空格。

```makefile
obj-m += lkm_example.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

如果我们运行 "make"，它将成功编译你编写的模块，编译后的文件为 "lkm_example.ko"，如果收到任何错误，请检查示例源文件中的引号是否正确，并且不要将其粘贴为UTF-8字符。

现在我们可以将此模块加载进内核进行测试了，命令如下：

```shell
sudo insmod lkm_example.ko
```

如果一切顺利，你将看不到任何输出，因为printk函数不会输出到控制台，而是输出到内核日志。要看到内核日志中的内容，我们需要运行：

```shell
sudo dmesg
```

你应该看到以时间戳为前缀的行："Hello, World!"，这意味着我们的内核模块已加载并成功打印到内核日志中。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201014161151.png" style="zoom:50%;" />

我们还可以检查模块是否已被加载：

```shell
lsmod | grep "lkm_example"
```

要卸载模块，运行：

```shell
sudo rmmod lkm_example
```

如果再次运行dmesg，你将看到"Goodbye, World!" 在日志中。你也可以再次使用lsmod命令确认它已卸载。

如你所见，此测试工作流程有点繁琐，因此要使其自动化，我们可以在Makefile中添加：

```makefile
test:
	sudo dmesg -C
	sudo insmod lkm_example.ko
	sudo rmmod lkm_example.ko
	dmesg
```

现在，运行：

```shell
make test
```

测试我们的模块并查看内核日志的输出，而不必运行单独的命令。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201014161249.png" style="zoom:50%;" />

现在，我们有了一个功能齐全，但又很简单的内核模块！



## 5. 一般模块

让我们再思考下。尽管内核模块可以完成各种任务，但与应用程序进行交互是其最常见的用途之一。

由于操作系统限制了应用程序查看内核空间内存的内容，因此，应用程序必须使用API与内核进行通信。尽管从技术上讲，有多种方法可以完成此操作，但最常见的方法是创建设备文件。

你以前可能已经与设备文件进行过交互。使用 /dev/zero，/dev/null 或类似设备的命令就是与名为 zero 和 null 的设备进行交互，这些设备将返回期望的值。

在我们的示例中，我们将返回 "Hello，World"，虽然这些字符串对于应用程序并没有什么用，但它将显示通过设备文件响应应用程序的过程。

这是完整代码：

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Robert W. Oliver II");
MODULE_DESCRIPTION("A simple example Linux module.");
MODULE_VERSION("0.01");

#define DEVICE_NAME "lkm_example"
#define EXAMPLE_MSG "Hello, World!\n"
#define MSG_BUFFER_LEN 15

/* Prototypes for device functions */
static int device_open(struct inode *, struct file *);
static int device_release(struct inode *, struct file *);
static ssize_t device_read(struct file *, char *, size_t, loff_t *);
static ssize_t device_write(struct file *, const char *, size_t, loff_t *);
               
static int major_num;
static int device_open_count = 0;
static char msg_buffer[MSG_BUFFER_LEN];
static char *msg_ptr;
               
/* This structure points to all of the device functions */
static struct file_operations file_ops = {
 .read = device_read,
 .write = device_write,
 .open = device_open,
 .release = device_release
};
               
/* When a process reads from our device, this gets called. */
static ssize_t device_read(struct file *flip, char *buffer, size_t len, loff_t *offset) {
	int bytes_read = 0;
 	/* If we’re at the end, loop back to the beginning */
 	if (*msg_ptr == 0) {
 		msg_ptr = msg_buffer;
 	}
 	/* Put data in the buffer */
 	while (len && *msg_ptr) {
   	/* Buffer is in user data, not kernel, so you can’t just reference
     * with a pointer. The function put_user handles this for us */
    put_user(*(msg_ptr++), buffer++);
    len--;
    bytes_read++;
	}
 	return bytes_read;
}
               
/* Called when a process tries to write to our device */
static ssize_t device_write(struct file *flip, const char *buffer, size_t len, loff_t *offset) {
	/* This is a read-only device */
 	printk(KERN_ALERT "This operation is not supported.\n");
 	return -EINVAL;
}
         
/* Called when a process opens our device */
static int device_open(struct inode *inode, struct file *file) {
 	/* If device is open, return busy */
 	if (device_open_count) {
 		return -EBUSY;
 	}
 	device_open_count++;
 	try_module_get(THIS_MODULE);
 	return 0;
}
         
/* Called when a process closes our device */
static int device_release(struct inode *inode, struct file *file) {
 	/* Decrement the open counter and usage count. Without this, the module would not unload. */
 	device_open_count--;
 	module_put(THIS_MODULE);
 	return 0;
}
         
static int __init lkm_example_init(void) {
 	/* Fill buffer with our message */
 	strncpy(msg_buffer, EXAMPLE_MSG, MSG_BUFFER_LEN);
 	/* Set the msg_ptr to the buffer */
 	msg_ptr = msg_buffer;
 	/* Try to register character device */
 	major_num = register_chrdev(0, "lkm_example", &file_ops);
 	if (major_num < 0) {
  	printk(KERN_ALERT "Could not register device: %d\n", major_num);
  	return major_num;
 	} else {
  	printk(KERN_INFO "lkm_example module loaded with device major number %d\n", major_num);
  	return 0;
 	}
}

static void __exit lkm_example_exit(void) {
 	/* Remember — we have to clean up after ourselves. Unregister the character device. */
 	unregister_chrdev(major_num, DEVICE_NAME);
 	printk(KERN_INFO "Goodbye, World!\n");
}

/* Register module functions */
module_init(lkm_example_init);
module_exit(lkm_example_exit);
```

既然我们的示例所做的不仅仅是在加载和卸载时打印一条消息，让我们修改Makefile，使其仅加载模块而不卸载模块：

```makefile
obj-m += lkm_example.o
all:
 	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
test:
  # We put a — in front of the rmmod command to tell make to ignore
  # an error in case the module isn’t loaded.
  -sudo rmmod lkm_example
  # Clear the kernel log without echo
  sudo dmesg -C
  # Insert the module
  sudo insmod lkm_example.ko
  # Display the kernel log
  dmesg
```

现在，当您运行 "make test" 时，您将看到设备主号码的输出。在我们的示例中，这是由内核自动分配的，但是，你需要此值来创建设备。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201014171608.png" style="zoom:50%;" />

获取从 "make test" 获得的值，并使用它来创建设备文件，以便我们可以从用户空间与内核模块进行通信：

```shell
sudo mknod /dev/lkm_example c MAJOR 0
```

在上面的示例中，将MAJOR替换为你运行 "make test" 或 "dmesg" 后得到的值，我得到的MAJOR为236，如上图，mknod命令中的 "c" 告诉mknod我们需要创建一个字符设备文件。

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201014171832.png" style="zoom:50%;" />

现在我们可以从设备中获取内容：

```shell
cat /dev/lkm_example
```

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201014172339.png" style="zoom:50%;" />

或者通过 "dd" 命令：

```shell
dd if=/dev/lkm_example of=test bs=14 count=100
```

你也可以通过应用程序访问此设备，它们不必编译应用程序--甚至Python、Ruby和PHP脚本也可以访问这些数据。

完成测试后，将其删除并卸载模块：

```shell
sudo rm /dev/lkm_example
sudo rmmod lkm_example
```



## 6. 结论

尽管我提供的示例是简单内核模块，但你完全可以根据此结构来构造自己的模块，以完成非常复杂的任务。

请记住，你在内核模块开发过程中完全靠自己。如果你为客户提供一个项目的报价，一定要把预期的调试时间增加一倍，甚至三倍。内核代码必须尽可能的完美，以确保运行它的系统的完整性和可靠性。



> 本文参考：https://blog.sourcerer.io/writing-a-simple-linux-kernel-module-d9dc3762c234
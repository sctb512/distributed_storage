# Linux内核模块开发（api通信）

通过上一篇文章[Linux内核模块开发（简单）](https://www.cnblogs.com/sctb/p/13816110.html)，我们已经实现了一个简单的内核模块，但是，更一般的内核模块需要提供api接口，方便用户态程序与其交互。



## 1. api通信原理







## 2. 测试程序

为了避免一个结构体在多个文件中重复定义，我把用于通信的结构体放在 lkm_example.h 头文件中，这样，内核模块可以引用该头文件，用户程序也可以。

lkm_example.h文件内容：

```c
//new
struct info {
        int num;
        char str[32];
};
```

内核模块 lkm_example.c 的内容：

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

#include <linux/fs.h>
#include <linux/uaccess.h>

#include <linux/device.h>
#include <linux/list.h>

#include "lkm_example.h"

MODULE_LICENSE("GPL");
MODULE_AUTHOR("abin");
MODULE_DESCRIPTION("A simple example Linux module.");
MODULE_VERSION("0.01");

#define DEVICE_NAME "lkm_example"
#define EXAMPLE_MSG "hello,abin!\n"
#define MSG_BUFFRE_LEN 15
//new
#define CLASS_NAME  "abin_module"

// declaration for function
static int device_open(struct inode *, struct file *);
static int device_release(struct inode *, struct file *);
static ssize_t device_read(struct file *, char *, size_t, loff_t *);
static ssize_t device_write(struct file *, const char *, size_t, loff_t *);
//new
static long device_ioctl(struct file*, unsigned int, unsigned long);

static int major_num;
static int device_open_count = 0;
static char msg_buffer[MSG_BUFFRE_LEN];
static char *msg_ptr;
//new
static struct class* device_class;
static struct device* device_;

// register function of read,write,open,release
static struct file_operations file_ops = {
        .read = device_read,
        .write = device_write,
        .open = device_open,
        .release = device_release,
        //new
        .unlocked_ioctl = device_ioctl
};

//new
#define BUF_LEN 32

// struct info {
//         int num;
//         char str[32];
// };

//new
static long device_ioctl(struct file *file, unsigned int cmd, unsigned long param) {
        switch (cmd) {
        case 0: {
                // char ch;
                struct info info_obj;
                struct info *tmp = (struct info *)param;
                // int i = 0;
                // printk(KERN_INFO "addr: %p", tmp);
                copy_from_user(&info_obj, tmp, sizeof(struct info));
                // for (i = 0 ; ch && i < BUF_LEN; i++, tmp++) {
                //         get_user(ch, tmp);
                //         printk(KERN_INFO "from user: %c", ch);
                // }
                printk(KERN_INFO "num: %d\n", info_obj.num);
                printk(KERN_INFO "str: %s\n", info_obj.str);
                printk(KERN_INFO "inner function (ioctl 0) finished.\n");
                info_obj.num = 1314;
                strcpy(info_obj.str, "i am fine, thank you.");
                copy_to_user(tmp, &info_obj, sizeof(struct info));
                break;
                }
        default:{
                printk(KERN_INFO "Unknown ioctl cmd.\n");
                break;
                }
        }
        return 0;
}

static ssize_t device_read(struct file *flip, char *buffer, size_t len, loff_t *offset) {
        int bytes_read = 0;
        if (*msg_ptr == 0) {
                msg_ptr = msg_buffer;
        }
        while (len && *msg_ptr) {
                put_user(*(msg_ptr++), buffer++);
                len--;
                bytes_read++;
        }
        return bytes_read;
}

static ssize_t device_write(struct file *flip, const char *buffer, size_t len, loff_t *offset) {
        printk(KERN_ALERT "this operate is not support.\n");
        return -EINVAL;
}

static int device_open(struct inode *inode, struct file *file) {
        if(device_open_count) {
                return -EBUSY;
        }

        device_open_count++;
        // i don't know what's meaning...
        try_module_get(THIS_MODULE);
        return 0;
}

static int device_release(struct inode *inode, struct file *file) {
        device_open_count--;
        module_put(THIS_MODULE);
        return 0;
}

static int __init lkm_example_init(void) {
        strncpy(msg_buffer, EXAMPLE_MSG, MSG_BUFFRE_LEN);
        msg_ptr = msg_buffer;
        // register character device
        major_num = register_chrdev(0, "lkm_example", &file_ops);
        if (major_num<0) {
                printk(KERN_ALERT "could not register device: %d\n", major_num);
                return major_num;
        }else{
                printk(KERN_INFO "lkm_example module loaded with device major number %d\n", major_num);
        }

        device_class = class_create(THIS_MODULE, CLASS_NAME);
        if (IS_ERR(device_class)){
                unregister_chrdev(major_num, DEVICE_NAME);
                printk(KERN_INFO "class device register failed!\n");
                return PTR_ERR(device_class);
        }
        printk(KERN_INFO "class device register success!\n");

        //最后，使用device_create函数注册设备驱动
        device_ = device_create(device_class, NULL, MKDEV(major_num, 0), NULL, DEVICE_NAME);
        if (IS_ERR(device_)){                           // Clean up if there is an error
                class_destroy(device_class);            // Repeated code but the alternative is goto statements
                unregister_chrdev(major_num, DEVICE_NAME);
                printk(KERN_ALERT "failed to create the device\n");
                return PTR_ERR(device_);
        }
        printk(KERN_INFO "create the device success!\n");

        return 0;
}

static void __exit lkm_example_exit(void) {
        unregister_chrdev(major_num, DEVICE_NAME);
        device_destroy(device_class, MKDEV(major_num, 0));
        class_destroy(device_class);
        printk(KERN_INFO "clean up successful. Bye.\n");
}

module_init(lkm_example_init);
module_exit(lkm_example_exit);
```

用户态程序，用于同内核模块api通信：

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

#include <string.h>

#include "lkm_example.h"

// struct info {
//         int num;
//         char str[32];
// };

int main()
{
        int fd;
        fd = open("/dev/lkm_example", O_RDWR);
        if (fd < 0)
        {
                perror("failed to open th device...\n");
                return errno;
        }else
        {
                printf("open device successful!\n");
        }
        struct info info_obj;
        info_obj.num = 520;
        strcpy(info_obj.str, "are you ok?");
				
  			//用户程序和内核模块通信的函数：ioctl()
        ioctl(fd, 0, (unsigned long)&(info_obj));
        printf("call ioctl with parameter 0!\n");
        printf("num: %d\n", info_obj.num);
        printf("num: %s\n", info_obj.str);
}
```



Makefile文件：

```makefile
obj-m += lkm_example.o
all:
	 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
c:
	 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

t:
	@clear
	@-sudo rmmod lkm_example
	@sudo dmesg -C
	@sudo insmod lkm_example.ko
	@dmesg
	@-sudo rm -rf /dev/lkm_example
	@sudo mknod /dev/lkm_example c 236 0

	@sudo dmesg -C
	@gcc main.c
	@echo ""
	@echo "> a.out"
	@sudo ./a.out
	@echo ""
	@echo "> ioctl"
	@dmesg
	@echo ""
```

其中， "@" 表示不输出命令内容，"-" 表示如果该行执行出错时不中断 make 过程，默认情况下，如果某一行执行出错，make会终止。



## 3. 结果分析

先运行 make 编译并生成内核模块，然后运行 "make t" 测试用户态程程序和内核模块api的通信:

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201015152647.png" style="zoom:50%;" />

通过输出结果，我们可以看到，内核模块收到了用户程序通过 ioctl() 函数传过来的结构体指针，并调用 copy_from_user()函数将该地址 sizeof(struct info) 长度的内容拷贝到局部变量 info_obj 中。

随后，内核模块将局部变量 info_obj 的属性设置为需要向用户态程序传递的值，调用 copy_to_user() 函数，将该结构体拷贝到从用户空间传来的结构体指针处。

这样，用户空间程序再次打印该结构体指针处的值时，已经编程内核模块修改后的内容。

 注意：上面截图中，看起来好像用户态程序在内核模块收到api请求之前就输出了被内核修改后的值。这是因为Makefile文件中，是在用户态程序 a.out 运行完成后，再执行 dmesg 查看内核的日志。实际上，内核模块打印日志的时间应该早于用户态程序输出的时间。



> 问：内核态是可以直接访问用户态的虚拟地址空间的，为什么还需要使用copy_from_user接口呢？ 
>
> 答：直接访问的话，无法保证被访问的用户态虚拟地址是否有对应的页表项，即无法保证该虚拟地址已经分配了相应的物理内存，如果此时没有对应的页表项，那么此时将产生page fault，导致流程混乱，原则上如果没有页表项(即没有物理内存时)，是不应该对其进行操作的。
>
> 参考：https://blog.csdn.net/u014353386/article/details/51033514


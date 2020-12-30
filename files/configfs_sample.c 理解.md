# configfs_sample.c 理解



## 1. 编译运行

代码从如下链接获得：

> [https://github.com/torvalds/linux/blob/master/samples/configfs/configfs_sample.c](https://github.com/torvalds/linux/blob/master/samples/configfs/configfs_sample.c)

编写 Makefile 文件：

```makefile
obj-m += configfs_sample.o
all:
	 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

编译生成内核模块：

```shell
make
ls -l
```

> -rwxr--r-- 1 abin abin 10K Oct 27 16:58 configfs_sample.c
>
> -rw-rw-r-- 1 abin abin 13K Oct 29 11:16 <font color="red">configfs_sample.ko</font>
>
> -rw-rw-r-- 1 abin abin 603 Oct 29 11:16 configfs_sample.mod.c
>
> -rw-rw-r-- 1 abin abin 2.6K Oct 29 11:16 configfs_sample.mod.o
>
> -rw-rw-r-- 1 abin abin 12K Oct 29 11:16 configfs_sample.o
>
> -rw-rw-r-- 1 abin abin 166 Oct 29 11:16 Makefile
>
> -rw-rw-r-- 1 abin abin  92 Oct 29 11:16 modules.order
>
> -rw-rw-r-- 1 abin abin  0 Oct 29 11:16 Module.symvers

其中，`configfs_sample.ko` 使编译好的内核模块，使用如下命令加载该模块：

```shell
sudo modprobe configfs_sample.ko
```

如果出现如下错误：

> modprobe: FATAL: Module configfs_sample.ko not found in directory <font color="red">/lib/modules/4.15.0-117-generic</font>

将 configfs_sample.ko 拷贝进 /lib/modules/4.15.0-117-generic 再次尝试。

查看 configfs_sample.ko 内核模块是否已经挂载：

```shell
lsmod  | grep configfs_sample
```

> <font color="red">configfs_sample</font>    16384 0

查看 configfs 根目录：

```shell
ls -l /sys/kernel/config/
```

> total 0
>
> drwxr-xr-x 2 root root 0 Oct 29 11:32 <font color="red">01-childless</font>
>
> drwxr-xr-x 2 root root 0 Oct 29 11:32 <font color="red">02-simple-children</font>
>
> drwxr-xr-x 2 root root 0 Oct 29 11:32 <font color="red">03-group-children</font>

如需卸载模块，使用如下命令：

```shell
sudo modprobe -r configfs_sample.ko
```



## 2. 代码理解

为了理解代码，我们首先整理一下 configfs 中的层级结构：

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201102155722.png)



内核模块初始化入口：

```c
module_init(configfs_example_init);
```

`configfs_example_init(void)` 函数：

```c
/*
 * 此处是configfs_subsystem结构体数组，分别对应示例中的三个configfs子系统
 */
static struct configfs_subsystem *example_subsys[] = {
  &childless_subsys.subsys,
  &simple_children_subsys,
  &group_children_subsys,
  NULL,
};

static int __init configfs_example_init(void)
{
  int ret;
  int i;
  struct configfs_subsystem *subsys;	//configfs子系统
  
  for (i = 0; example_subsys[i]; i++) {
    subsys = example_subsys[i];
    
    config_group_init(&subsys->su_group);				//初始化 group
    mutex_init(&subsys->su_mutex);							//初始化 mutex
    ret = configfs_register_subsystem(subsys);	//注册 subsystem
    if (ret) {
      printk(KERN_ERR "Error %d while registering subsystem %s\n", ret, subsys->su_group.cg_item.ci_namebuf);
      goto out_unregister;
    }
  }
  
  return 0;

out_unregister:
  for (i--; i >= 0; i--)
    configfs_unregister_subsystem(example_subsys[i]);
  
  return ret;
}
```

程序的主要逻辑是通过 struct configfs_subsystem 结构体传递给 configfs 的，下面分别对3个示例进行分析。

### 2.1 示例01-childless

变量 childless_subsys 的内容：

```c
struct childless {
  struct configfs_subsystem subsys;
  int showme;
  int storeme;
};

static struct childless childless_subsys = {
  .subsys = {
    .su_group = {
      .cg_item = {
        .ci_namebuf = "01-childless",
        .ci_type = &childless_type,		//struct config_item_type，定义操作、属性等
      },
    },
  },
};
```

childless_type 变量如下：

```c
static const struct config_item_type childless_type = {
  .ct_attrs	= childless_attrs,		//configfs_attribute，只定义了属性，没有定义对item和group操作
  .ct_owner	= THIS_MODULE,
};
```

childless_attrs 是一个数组，以 NULL 结尾。以下定义了三个属性，在 configfs 中，将表现为3个文件：

```c
static struct configfs_attribute *childless_attrs[] = {
  &childless_attr_showme,
  &childless_attr_storeme,
  &childless_attr_description,
  NULL,
};
```

childless_attr_showme，childless_attr_storeme 和 childless_attr_description 三个属性是通过以下函数创建的：

```c
CONFIGFS_ATTR_RO(childless_, showme);	//需要定义childless_showme_show()函数
CONFIGFS_ATTR(childless_, storeme);		//需要定义childless_storeme_show()和childless_storeme_store()函数
CONFIGFS_ATTR_RO(childless_, description);	//需要定义childless_description_show()函数
```

创建属性的函数有3个，在 <font color="green">linux/configfs.h</font> 中：

```c
#define CONFIGFS_ATTR(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
    .ca_name	= __stringify(_name),		\
    .ca_mode	= S_IRUGO | S_IWUSR,		\
    .ca_owner	= THIS_MODULE,			\
    .show		= _pfx##_name##_show,		\
    .store		= _pfx##_name##_store,		\
}

#define CONFIGFS_ATTR_RO(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
    .ca_name	= __stringify(_name),		\
    .ca_mode	= S_IRUGO,			\
    .ca_owner	= THIS_MODULE,			\
    .show		= _pfx##_name##_show,		\
}

#define CONFIGFS_ATTR_WO(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
    .ca_name	= __stringify(_name),		\
    .ca_mode	= S_IWUSR,			\
    .ca_owner	= THIS_MODULE,			\
    .store		= _pfx##_name##_store,		\
}
```

可以看到，这三个宏定义函数可以根据传入的参数定义不同的结构体变量，变量名为：<font color="red">\_pfx</font>_attr\_<font color="red">name</font>，同时也会定义相应的 show 和 store 函数名。

CONFIGFS_ATTR(_pfx, _name) 需要定义 show 和 store 函数，相应的函数名分别为：<font color="red">\_pfx\_name</font>\_show 和 <font color="red">\_pfx\_name</font>\_store；

CONFIGFS_ATTR_RO(_pfx, _name)只需要定义 show 函数；

CONFIGFS_ATTR_WO(_pfx, _name) 只需要定义 store 函数。

childless_showme_show()，childless_storeme_show()，childless_storeme_store()和childless_description_show()的定义如下：

```c
/*
 * 传入item，得到该item所在的childless结构体
 */
static inline struct childless *to_childless(struct config_item *item)
{
  return item ? container_of(to_configfs_subsystem(to_config_group(item)), struct childless, subsys) : NULL;
}

//childless_showme_show函数的实现，根据item找到结构体struct childless，输出childless->showme，然后将childless->showme加1
static ssize_t childless_showme_show(struct config_item *item, char *page)
{
  struct childless *childless = to_childless(item);
  ssize_t pos;

  pos = sprintf(page, "%d\n", childless->showme);
  childless->showme++;

  return pos;
}
//childless_storeme_show函数实现，输出结构体struct childless成员storeme的值
static ssize_t childless_storeme_show(struct config_item *item, char *page)
{
  return sprintf(page, "%d\n", to_childless(item)->storeme);
}
//childless_storeme_store函数实现，接受从文件系统输入的值，保存在struct childless成员storeme中
static ssize_t childless_storeme_store(struct config_item *item, const char *page, size_t count)
{
  struct childless *childless = to_childless(item);
  unsigned long tmp;
  char *p = (char *) page;

  tmp = simple_strtoul(p, &p, 10);		//将字符串转化为10进制数字，类型为unsigned long
  if (!p || (*p && (*p != '\n')))
    return -EINVAL;

  if (tmp > INT_MAX)
    return -ERANGE;

  childless->storeme = tmp;

  return count;
}
//childless_description_show函数实现，向page中填充内容
static ssize_t childless_description_show(struct config_item *item, char *page)
{
  return sprintf(page,
                 "[01-childless]\n"
                 "\n"
                 "The childless subsystem is the simplest possible subsystem in\n"
                 "configfs.  It does not support the creation of child config_items.\n"
                 "It only has a few attributes.  In fact, it isn't much different\n"
                 "than a directory in /proc.\n");
}
```

根据我的理解，page指向一块内存空间，这块空间接收来自文件系统的数据，同时，负责将configfs中的内容输出给文件系统。

showme 文件运行效果：

```shell
cat showme
```

> 1

```shell
cat showme
```

> 2

storeme 文件运行效果：

```shell
cat storeme
```

> 0

```shell
echo 1111 > storeme
cat storeme
```

> 1111

### 2.2 示例02-simple-children

simple_children_subsys 变量的内容：

```c
struct simple_children {
  struct config_group group;
};

static struct configfs_subsystem simple_children_subsys = {
  .su_group = {
    .cg_item = {
      .ci_namebuf = "02-simple-children",
      .ci_type = &simple_children_type,
    },
  },
};
```

simple_children_type 内容：

```c
static const struct config_item_type simple_children_type = {
  .ct_item_ops	= &simple_children_item_ops,		//item的操作
  .ct_group_ops	= &simple_children_group_ops,		//group的操作
  .ct_attrs	= simple_children_attrs,	//属性，和01相同
  .ct_owner	= THIS_MODULE,
};
```

可以看到，与01示例相比，02-siimple-chiildren 不光定义了 ct_attrs，还定义了 ct_item_ops 和 ct_group_ops。先看看赋值给 ct_attrs 的变量 simple_children_attrs：

```c
static struct configfs_attribute *simple_children_attrs[] = {
  &simple_children_attr_description,		//属性，configfs中表示为文件
  NULL,
};
```

simple_children_attrs 定义对象的属性，在 configfs 中表示为文件。simple_children_attr_description 是通过宏函数创建的：

```c
CONFIGFS_ATTR_RO(simple_children_, description);
```

在 CONFIGFS_ATTR_RO 宏函数中会使用 show 函数，定义如下：

```c
static ssize_t simple_children_description_show(struct config_item *item, char *page)
{
  return sprintf(page,
                 "[02-simple-children]\n"
                 "\n"
                 "This subsystem allows the creation of child config_items.  These\n"
                 "items have only one attribute that is readable and writeable.\n");
}
```

此处和01示例没什么区别，主要看 ct_item_ops 和 ct_group_ops。<font color="red">simple_children_item_ops</font> 的定义如下：

```c
static struct configfs_item_operations simple_children_item_ops = {
  .release	= simple_children_release,		//实现release函数
};
```

simple_children_item_ops 是 struct configfs_item_operations 类型，也很简单，只定义了 release 函数，simple_children_release 函数定义如下：

```c
static void simple_children_release(struct config_item *item)
{
  kfree(to_simple_children(item));	//将item转换为simple_children结构体并释放内核分配的内存
}
```

<font color="red">simple_children_group_ops</font> 的定义如下：

```c
static struct configfs_group_operations simple_children_group_ops = {
  .make_item	= simple_children_make_item,
};
```

simple_children_group_ops 也很简单，只实现了 make_item 函数。simple_children_make_item 如下：

```c
/*
 * 传入item，得到该item所在的simple_children结构体
 */
static inline struct simple_children *to_simple_children(struct config_item *item)
{
  return item ? container_of(to_config_group(item), struct simple_children, group) : NULL;
}

static struct config_item *simple_children_make_item(struct config_group *group, const char *name)
{
  struct simple_child *simple_child;

  simple_child = kzalloc(sizeof(struct simple_child), GFP_KERNEL);	//为simple_child分配内存
  if (!simple_child)
    return ERR_PTR(-ENOMEM);

  config_item_init_type_name(&simple_child->item, name, &simple_child_type);	//创建新的item时，使用config_item_init_type_name初始化，simple_child_type是子item使用的config_item_type结构体

  simple_child->storeme = 0;	//将simple_child的storeme设置为0

  return &simple_child->item;
}
```

simple_child_type定义如下：

```c
struct simple_child {
  struct config_item item;
  int storeme;
};

static const struct config_item_type simple_child_type = {
  .ct_item_ops	= &simple_child_item_ops,
  .ct_attrs	= simple_child_attrs,
  .ct_owner	= THIS_MODULE,
};
```

同上，定义了 ct_attrs 和 ct_item_ops，没有定义 ct_item_ops，simple_child_attrs变量定义如下：

```c
CONFIGFS_ATTR(simple_child_, storeme);

static struct configfs_attribute *simple_child_attrs[] = {
  &simple_child_attr_storeme,
  NULL,
};
```

需要定义 show 和 store 函数：

```c
static inline struct simple_child *to_simple_child(struct config_item *item)
{
  return item ? container_of(item, struct simple_child, item) : NULL;
}

/*
 * 子item的show函数，将item转换为simple_child结构体并输出storeme的值
 */
static ssize_t simple_child_storeme_show(struct config_item *item, char *page)
{
  return sprintf(page, "%d\n", to_simple_child(item)->storeme);
}

/*
 * 子item的store函数，将从文件系统输入的值保存在simple_child->storeme中
 */
static ssize_t simple_child_storeme_store(struct config_item *item, const char *page, size_t count)
{
  struct simple_child *simple_child = to_simple_child(item);
  unsigned long tmp;
  char *p = (char *) page;

  tmp = simple_strtoul(p, &p, 10);	//将字符串转换为10进制数字
  if (!p || (*p && (*p != '\n')))
    return -EINVAL;

  if (tmp > INT_MAX)
    return -ERANGE;

  simple_child->storeme = tmp;

  return count;
}
```

运行效果：

```shell
make child
ls
```

> <font color="red">child</font>  description

```shell
cd child
ls -l
```

> total 0
> -rw-r--r-- 1 root root 4096 Nov  3 21:57 <font color="red">storeme</font>

child 文件夹中的 store 文件是自动创建的，这是因为定义了 make_item 函数，初始化 item 时 simple_child_type 变量的作用。

```shell
cat storeme
```

> 0

```shell
echo 2222 > storeme
cat storeme
```

> 2222

### 2.3 示例03-group-children

group_children_subsys变量的内容为：

```c
static struct configfs_subsystem group_children_subsys = {
  .su_group = {
    .cg_item = {
      .ci_namebuf = "03-group-children",
      .ci_type = &group_children_type,
    },
  },
};
```

group_children_type 的定义如下：

```c
static const struct config_item_type group_children_type = {
  .ct_group_ops	= &group_children_group_ops,
  .ct_attrs	= group_children_attrs,
  .ct_owner	= THIS_MODULE,
};
```

可以看到，和示例02相比，此处没有定义对 item 的操作，只定义了对 group 的操作。先看 group_children_attrs 的定义：

```c
CONFIGFS_ATTR_RO(group_children_, description);

static struct configfs_attribute *group_children_attrs[] = {
  &group_children_attr_description,
  NULL,
};
```

同样，使用 CONFIGFS_ATTR_RO 宏定义函数需要先定义好 show 函数：

```c
static ssize_t group_children_description_show(struct config_item *item, char *page)
{
  return sprintf(page,
                 "[03-group-children]\n"
                 "\n"
                 "This subsystem allows the creation of child config_groups.  These\n"
                 "groups are like the subsystem simple-children.\n");
}
```

此处和示例01和02都一样，下面是示例03的重点：

```c
static struct config_group *group_children_make_group( struct config_group *group, const char *name)
{
  struct simple_children *simple_children;	//此处使用的struct simple_children结构体是示例02中定义的结构体

  simple_children = kzalloc(sizeof(struct simple_children), GFP_KERNEL);	//分配内存
  if (!simple_children)
    return ERR_PTR(-ENOMEM);

  config_group_init_type_name(&simple_children->group, name, &simple_children_type);	//初始化group，simple_children_type也是示例02中定义的

  return &simple_children->group;
}
```

这段代码负责创建group，初始化group时使用 simple_children_type变量，该变量是 struct config_item_type 类型，其中定义的内容就是实例02的内容。

在configfs中的表现为：在 03-group-children 下创建的每个目录，都相当于加载内核模快时创建的 02-simple-children 目录。

运行效果：

```shell
mkdir group
ls
```

> description  <font color="red">group</font>

```shell
cd group
ls -l
```

> total 0
> -r--r--r-- 1 root root 4096 Nov  3 22:20 description

```shell
cat description
```

> <font color="red">[02-simple-children]</font>
>
> This subsystem allows the creation of child config_items.  These
> items have only one attribute that is readable and writeable.

到这里可以看到，在示例03创建的目录等同于 02-simple-children 目录，下面的操作的示例02效果一样。

```shell
mkdir group_child
ls
```

> description  <font color="red">group_child</font>

```shell
cd group_child
ls
```

> storeme

```shell
cat storeme
```

> 0

``` shell
echo 3333 > storeme
cat storeme
```

> 3333
# configfs-用户空间控制的内核对象配置

![](https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201101221320.png)

## 1. 什么是configfs? 

configfs 是一个基于内存的文件系统，它提供了与sysfs相反的功能。sysfs 是一个基于文件系统的内核对象视图，而configfs 是一个基于文件系统的内核对象管理器（或称为config_items）。

在 sysfs 中，一个对象在内核中被创建（例如，当内核发现一个设备时），并在 sysfs 中注册，然后它的属性会出现在 sysfs 中，允许用户空间通过 readdir(3)/read(2) 读取，同时，也允许用户通过 write(2) 修改一些属性。很重要的一点，对象是在内核中被创建和销毁的，内核控制着 sysfs 表示内核对象的生命周期，而 sysfs 不能干预。

configfs 的 config_item 是通过用户空间显式操作 mkdir(2)  创建， rmdir(2) 销毁的。对象的属性在 mkdir(2) 时出现，并且可以通过 read(2) 和 write(2) 进行读取或修改。和 sysfs 一样，readdir(3) 可以查询 items 和属性的列表，symlink(2) 可以用来将 items 分组。与 sysfs 不同的是，configfs 表示的生命周期完全由用户空间控制，支持这些 items 的内核模块必须对用户控制做出响应。

sysfs 和 configfs 应该同时存在于一个系统中，任何一个都不能取代另一个。

 

## 2. 使用configfs 

configfs 可以编译为一个模块，也可以编译到内核中。你可以通过以下命令访问它：

```shell
sudo mount -t configfs none /sys/kernel/config/
```

除非客户端模块也被加载，否则 configfs 树是空的。这些模块将它们的 item 类型作为子系统注册到 configfs 中，一旦客户端子系统被加载，它将作为一个（或多个）子目录出现在 /sys/kernel/config/ 下。和 sysfs 一样，无论是否挂载在 /sys/kernel/config/ 上，configfs树始终存在。

通过 mkdir(2) 创建一个 item。此时，item 的属性也会出现，readdir(3) 可以查看有哪些属性，read(2) 可以查询它们的默认值，write(2) 可以存储新的值。

<font color="red">注意</font>：不要在一个属性文件中存入多个属性。

**configfs 属性的两种类型：** 

1. 一般属性，与 sysfs 属性类似，是小型 ASCII 文本文件，最大尺寸为一页（ PAGE_SIZE ，在 i386 上为 4096 ）。每个文件最好只使用一个值，与 sysfs 的注意事项相同。

   configfs 希望 write(2) 能一次性存储整个缓冲区。在写入一般的 configfs 属性时，用户空间进程应该先读取整个文件，修改想要修改的部分，然后再把整个缓冲区写回去。

2. 二进制属性，与 sysfs 二进制属性有些类似，但在语义上做了一些轻微的改变。不受 PAGE_SIZE 的限制，但整个二进制 item 必须适合于单个内核 vmalloc 分配的buffer。

   用户空间的 write(2) 调用是有缓冲的，属性的 write_bin_attribute 方法会在其被关闭时调用，因此，用户空间必须检查 close(2) 的返回码，以便验证操作是否成功完成。

   为了避免恶意用户对内核进行OOMing（ "out of memory"，溢出攻击），每个二进制属性都有一个最大的缓冲区值。

当一个 item 需要被销毁时，用 rmdir(2) 删除它。如果一个 item 与（通过symlink(2)）其他 item 有链接，则不能销毁该 item。可以通过 unlink(2) 取消链接。

 

## 3. 配置FakeNBD：一个例子

想象一下，有一个网络块设备（NBD）驱动程序，它允许你访问远程块设备，我们称它为FakeNBD。FakeNBD 使用 configfs 进行配置，显然，需要提供一个很好用的用户态程序，让系统管理员能够方便地配置 FakeNBD，为了使对 FakeNBD 的配置起作用，这个程序必须将配置的信息告诉驱动，这就是 configfs 的作用。

当加载 FakeNBD 驱动时，它会在 configfs 中注册自己，用户能使用 readdir(3) 看到它。

```shell
ls /sys/kernel/config
```

>  fakenbd

用户也可以使用 mkdir(2) 创建 fakenbd 连接，名字是任意的。不过，示例中的名字可能已经被其他工具（uuid 或者磁盘名）使用了。

```shell
mkdir /sys/kernel/config/fakenbd/disk1
ls /sys/kernel/config/fakenbd/disk1
```

> target device rw

target 属性包含 FakeNBD 要连接的服务器的IP地址，device 属性是服务器上的设备，可想而知，rw属性决定了连接是只读还是读写。

 ```shell
echo 10.0.0.1 > /sys/kernel/config/fakenbd/disk1/target
echo /dev/sda1 > /sys/kernel/config/fakenbd/disk1/device
echo 1 > /sys/kernel/config/fakenbd/disk1/rw
 ```

就是这样，通过 shell 就已经把设备配置好了。

 

## 4. 用 configfs 编程

configfs 中的每个对象都是一个 config_item，config_item 就是子系统中的一个对象，它的属性与该对象上的值相匹配。configfs 处理该对象及其属性的文件系统表示，允许子系统忽略除基本 show/store 之外的其它所有交互。

items 是在 config_group 里面创建和销毁的。一个 group 是<font color="green">共享相同属性和操作</font>的 items 集合。items 由mkdir(2)创建，rmdir(2)删除，这些 configfs 都会处理， group中 有一组操作来执行这些任务。

子系统是客户端模块的顶层。在初始化过程中，客户端模块向 configfs 注册子系统，子系统作为一个目录出现在 configfs 文件系统的最高层（根）。一个子系统也是一个  config_group，可以做所有 config_group 能做的事情。 

### 4.1 config_item 结构体

```c
struct config_item {
    char                    *ci_name;
    char                    ci_namebuf[UOBJ_NAME_LEN];
    struct kref             ci_kref;
    struct list_head        ci_entry;
    struct config_item      *ci_parent;
    struct config_group     *ci_group;
    struct config_item_type *ci_type;
    struct dentry           *ci_dentry;
};

void config_item_init(struct config_item *);
void config_item_init_type_name(struct config_item *, const char *name, struct config_item_type *type);
struct config_item *config_item_get(struct config_item *);
void config_item_put(struct config_item *);
```

一般来说，config_item 结构体被嵌入到一个 container 结构体中，这个 container 结构体实际上代表了子系统正在做的事情，其中的 config_item 部分就是对象与 configfs 的交互方式。

无论是在源文件中静态定义还是由父 config_group 创建，创建 config_item 都必须调用一个 _init() 函数，这将初始化引用计数器并设置相应的字段。

config_item 的所有用户都应该通过 config_item_get() 引用它，并在完成后通过 config_item_put() 函数放弃这个引用。 

就其本身而言，config_item 只能在 configfs 中出现。通常，一个子系统希望这个 item 能够显示和存储属性，并完成一些其他事情，为此，还需要一个 type 结构体。

> 换句话说，config_item_type 结构体主要用来完成除了显示和存储属性之外的其他事情。

### 4.2 config_item_type 结构体

```c
struct configfs_item_operations {
    void (*release)(struct config_item *);
    int (*allow_link)(struct config_item *src, struct config_item *target);
    void (*drop_link)(struct config_item *src, struct config_item *target);
};

struct config_item_type {
    struct module                           *ct_owner;
    struct configfs_item_operations         *ct_item_ops;
    struct configfs_group_operations        *ct_group_ops;
    struct configfs_attribute               **ct_attrs;
    struct configfs_bin_attribute						**ct_bin_attrs;
};
```

config_item_type 最基本的功能是定义可以对 config_item 进行哪些操作。所有被动态分配的 item 都需要提供 ct_item_ops->release() 方法。当 config_item 的引用计数为零时，就会调用这个方法释放它。

### 4.3 configfs_attribute结构体 

```c
struct configfs_attribute {
    char                    *ca_name;
    struct module           *ca_owner;
    umode_t                 ca_mode;
    ssize_t (*show)(struct config_item *, char *);
    ssize_t (*store)(struct config_item *, const char *, size_t);
}; 
```

当一个 config_item 希望一个属性以文件的形式出现在项目的 configfs 目录中时，它必须定义一个 configfs_attribute 来描述它。然后，它将属性添加到以 NULL 结尾的数组 config_item_type->ct_attrs 中。当 item 出现在 configfs 中时，属性文件将以configfs_attribute->ca_name 文件名出现，configfs_attribute->ca_mode 指定文件权限。

如果一个属性是可读的，并且提供了一个 ->show 方法，那么每当用户空间要求对该属性进行 read(2) 时，该方法（ ->show ）就会被调用。如果一个属性是可写的，并且提供了一个 ->store 方法，那么每当用户空间要求对该属性进行 write(2) 时，该方法（ ->store ）就会被调用。

### 4.4 configfs_bin_attribute 结构体

 ```c
struct configfs_bin_attribute {
    struct configfs_attribute	 cb_attr;
    void											 *cb_private;
    size_t										 cb_max_size;
};
 ```

当需要使用二进制blob来显示 item 对应 configfs 目录中文件的内容时，，会使用二进制属性。

> BLOB：binary large object，二进制大对象，是一个可以存储二进制文件的容器。

将二进制属性添加到以 NULL 结尾的数组 config_item_type->ct_bin_attrs 中，item 就会出现在 configfs 中。属性文件会以 configfs_bin_attribute->cb_attr.ca_name 作为文件名， configfs_bin_attribute->cb_attr.ca_mode 指定文件权限。

cb_private 成员是提供给驱动程序使用的，cb_max_size 成员则指定了 vmalloc 缓冲区的最大可用空间。

如果二进制属性是可读的，并且 config_item 提供了 ct_item_ops->read_bin_attribute() 方法，那么每当用户空间要求对属性进行 read(2) 时，该方法就会被调用。同理，用户空间的 write(2) 操作会调用 ct_item_ops->write_bin_attribute() 方法。读/写会被缓冲，所以只会执行读/写的一个，属性本身不需要关心。

 ### 4.5 config_group 结构体 

config_item 不能凭空产生，唯一的方法是通过 mkdir(2) 在 config_group 上创建一个，该操作将触发子 item 的创建。

```c
struct config_group {
    struct config_item		cg_item;
    struct list_head		cg_children;
    struct configfs_subsystem 	*cg_subsys;
    struct list_head		default_groups;
    struct list_head		group_entry;
};

void config_group_init(struct config_group *group);
void config_group_init_type_name(struct config_group *group, const char *name, struct config_item_type *type);
```

config_group 结构包含一个 config_item，正确地配置该 item 意味着一个 group 可以单独作为一个 item。

此外，group 还可以完成更多工作：创建 item 或 group，这是通过在 group 中 config_item_type 指定的 group 操作来实现的。

```c
struct configfs_group_operations {
    struct config_item *(*make_item)(struct config_group *group, const char *name);
    struct config_group *(*make_group)(struct config_group *group, const char *name);
    int (*commit_item)(struct config_item *item);
    void (*disconnect_notify)(struct config_group *group, struct config_item *item);
    void (*drop_item)(struct config_group *group, struct config_item *item);
};
```

一个 group 通过提供 ct_group_ops->make_item() 方法来创建子项目。如果提供了这个方法，当在 group 目录中使用 mkdir(2) 时，该方法被调用。当 ct_group_ops->make_item() 方法被调用，子系统将分配一个新的 config_item（ or 更可能是它的 container 结构体），初始化并将其返回给 configfs，然后，configfs 将填充文件系统树以反映新的 item。

如果子系统希望子 item 本身是一个 group，子系统提供 ct_group_ops->make_group()，其他的操作都是一样，使用 group 上的 group _init() 函数初始化。

最后，当用户空间对 item 或 group 调用 rmdir(2) 时，会调用 ct_group_ops->drop_item() 方法。由于 config_group 也是一个 config_item，所以不需要单独的 drop_group() 方法。子系统必须调用 config_item_put() 函数释放 item 分配时初始化的引用，如果除了该操作，子系统不需要要做其它操作，可以省略 ct_group_ops->drop_item() 方法，configfs 将代表子系统对 item 调用 config_item_put() 方法。 

<font color="red">重要</font>：drop_item() 的返回值为 void，因此不能失败。当 rmdir(2) 被调用时，configfs 将会从文件系统树中删除该 item（假设没有子 item 正在使用它），子系统负责对此操作做出响应。如果子系统在其他线程中有对该 item 的引用，那么内存是安全的，该 item 从子系统中真正消失可能还需要一段时间，但它已经从 configfs 中消失了。

当 drop_item() 被调用时，item 的链接已经被拆掉了，它在父 item 上不再有引用，在 item 的层次结构中也没有位置。如果客户端需要在这个拆分发生之前做一些清理工作，子系统可以实现 ct_group_ops->disconnect_notify() 方法。该方法在 configfs 从文件系统结构中删除 item 后，item 从父 group 中删除前被调用，和drop_item()一样，disconnect_notify() 的返回值也为 void，不能失败。客户端子系统不应该在这里删除任何引用，因为这必须在 drop_item() 中进行。

当一个 config_group 还有子 item 的时候，它是不能被删除的，这在 configfs 的 rmdir(2) 代码中有实现。->drop_item() 不会被调用，因为该 item 没有被删除，rmdir(2) 也将失败，因为目录不是空的。

### 4.6 configfs_subsystem 结构体

一个子系统必须注册自己，通常是在 module_init 的时候，该操作告诉 configfs 让子系统出现在文件树中。

```c
struct configfs_subsystem {
    struct config_group	su_group;
    struct mutex		su_mutex;
};

int configfs_register_subsystem(struct configfs_subsystem *subsys);
void configfs_unregister_subsystem(struct configfs_subsystem *subsys);
```

一个子系统由一个顶级 config_group 和一个 mutex 组成，这个 group 是创建子 config_items 的地方。对于一个子系统，这个 group 通常是静态定义的，在调用 configfs_register_subsystem() 之前，子系统必须通过 group _init() 函数来初始化这个 group，并且还必须初始化 mutex。

当调用注册函数返回后，子系统会一直存在，并且可以在 configfs 中看到。这时，用户程序可以调用 mkdir(2)，子系统必须为此做好准备。



## 5. 一个例子

理解这些基本概念的最好例子是 <font color = "green">samples/configfs/configfs_sample.c</font> 中的 simple_children subsystem/group 和 simple_child item，它们展示了一个显示和存储属性的简单对象，以及一个创建和销毁这些子 item 的简单 group。

> configfs_sample.c : [https://github.com/torvalds/linux/blob/master/samples/configfs/configfs_sample.c](https://github.com/torvalds/linux/blob/master/samples/configfs/configfs_sample.c)



## 6. 层次导航和子系统互斥

configfs 还提供了一些额外的功能。由于 config_groups 和 config_items 出现在文件系统中，所以它们被安排在一个层次结构中。一个子系统是绝对不会接触到文件系统部分的，但是子系统可能会对这个层次结构感兴趣。出于这个原因，层次结构是通过 config_group->cg_children 和 config_item->ci_parent 结构体成员表示的。

子系统可以浏览 cg_children 列表和 ci_parent 指针来查看子系统创建的树。这可能会与 configfs 对层次结构的管理发生冲突，所以 configfs 使用子系统的 mutex 来保护修改。无论何时子系统要浏览层次结构，都必须在子系统 mutex 的保护下进行。

当一个新分配的 item 还没有被链接到这个层次结构中时，子系统将无法获得 mutex， 同样，当一个正在被删除的 item 还没有解除链接时，子系统也无法获取mutex。这意味着，当一个 item 在 configfs 中时，项目的 ci_parent 指针永远不会是 NULL，而且，同一时刻，item 只会存在一个父 item 的 cg_children 列表中，这允许子系统在持有 mutex 时信任 ci_parent 和 cg_children。 



## 7. 通过symlink(2)进行item汇总

configfs 通过 group->item 为父/子关系提供了一个简单的 group，但是，通常情况下，在更大的环境中需要在父/子关系之外进行聚合，这是通过 symlink(2) 实现的。

一个 config_item 可以提供 ct_item_ops->allow_link() 和 ct_item_ops->drop_link() 方法。如果 ->allow_link() 方法存在，就可以调用 symlink(2)，将 config_item 作为链接的来源。这些链接只允许在 configfs 的 config_items 之间进行，任何在 configfs 文件系统之外的 symlink(2) 调用都会被拒绝。 

当 symlink(2) 被调用时，源 config_item 的 ->allow_link() 方法会被自己和一个目标 item 调用，如果源 item 允许链接到目标 item，则返回0，如果源 item 只想链接到某一类型的对象（例如，在它自己子系统中的对象），它可以拒绝该链接。

当对符号链接调用 unlink(2) 时，通过 ->drop_link() 方法通知源 item，和 ->drop_item() 方法一样，这也是一个返回值为 void 的函数，不能失败，子系统负责响应因该函数执行导致的变化。

当一个 config_item 链接到任何其它 item 时，它不能被删除，当一个 item 链接到它时，也不能被删除。在 configfs 中不允许使用软链接。



 ## 8. 自动创建分组

一个新的 config_group 可能希望有两种类型的子 config_items，虽然这可以通过在 ->make_item() 中的 magic names 来编写，但更显式的方法是让用户空间能够看到这种不同。

configfs 提供了一种方法，即在创建父 group 时，在其内部自动创建一个或多个子 group，而不是把行为互不相同的 item 放在同一个 group 中。因此，mkdir("parent") 的结果是 "parent"，"parent/subgroup1"，直到 "parent/subgroupN"。现在，type 1 的 item 可以在目录 "parent/subgroup1" 中创建，type N 的 item 可以在目录 "parent/subgroupN" 中创建。

这些自动创建的子 group，或者说默认 group，并不影响父 group 的其他子 group，如果 ct_group_ops->make_group() 存在，其他子 group 也可以直接在父 group 上创建。

configfs 子系统通过 configfs_add_default_group() 函数将默认 group 添加到父 config_group 结构体中来指定它们，每个添加的 group 与父 group 同时被填充到 configfs 树中。同样地，它们也会与父 group 同时被删除，不会另外通知，当一个 ->drop_item() 方法调用通知子系统其父 group 即将消失时，意味着与该父 group 关联的每个默认子 group 也即将消失。

因此，不能直接通过 rmdir(2) 来删除默认 group，当父 group 的 rmdir(2) 检查子 group 时，也不会考虑它们（默认 group）。

 ## 9. 附属子系统

有时，某些驱动程序依赖于特定的 configfs item，例如，挂载 ocfs2 依赖于心跳区域 item，如果使用 rmdir(2) 删除该区域 item，则 ocfs2 挂载会出错 或转为 readonly 模式。

configfs 提供了两个额外的 API 调用：configfs_depend_item() 和 configfs_undepend_item()，客户端驱动程序可以在一个现有的 item 上调用 configfs_depend_item() 来告诉 configfs 它是被依赖的。如果其他程序 rmdir(2) 该 item，configfs 将返回 -EBUSY，当这个 item 不再被依赖时，客户端驱动会调用 configfs_undepend_item() 取消依赖。

这些 API 不能在任何的 configfs 回调下调用，因为它们会冲突，不过，它们可以阻塞和重分配。客户端驱动不能凭自己的直觉调用它们，它应该提供一个外部子系统调用的 API。

这是如何工作的呢？想象一下 ocfs2 的挂载过程。当它挂载时，它会要求一个心跳区域  item，这是通过对心跳代码的调用来完成的。在心跳代码中，区域 item 被查找出来，同时，心跳代码会调用 configfs_depend_item()，如果成功了，那么心跳代码就知道这个区域是安全的，可以交给 ocfs2，如果失败了，ocfs2 将被卸载，心跳代码优雅地传递出一个错误。

 ## 10. 可提交 item

<font color="red">注</font>：可提交的 item 目前尚未使用。

有些 config_item 不能有一个有效的初始状态，也就是说，不能为 item 的属性指定默认值（指定了默认值， item 才能起作用），用户空间必须配置一个或多个属性后，子系统才可以启动这个 item 所代表的实体。

考虑一下上面的 FakeNBD 设备，如果没有目标地址和目标设备，子系统就不知道要导入什么块设备。这个例子假设子系统只是简单地等待，直到所有属性都配置好了，再开始连接。每次属性存储操作都检查属性是否被初始化的方法确实可行，但这会导致在满足条件（属性都已经初始化）的情况下，每次属性存储操作必定触发连接。

更好的做法是用一个显式的操作来通知子系统 config_item 已经准备好了。更重要的是，显式操作允许子系统提供反馈，说明属性是否以合理的方式被初始化，configfs 以<font color="red">可提交 item</font> （commitable item）的形式提供了这种反馈。

configfs 仍然只使用正常的文件系统操作，通过 rename(2) 提交的一个item，会从一个可修改的目录移动到一个不能修改的目录。

任何提供 ct_group_ops->commit_item() 方法的 group 都有可提交 item，当这个 group 出现在 configfs 中时，mkdir(2) 将不会直接在该 group 中工作，相反，该 group 将有两个子目录 "live" 和 "pending"，live" 目录不支持 mkdir(2) 或 rmdir(2) ，它只允许 rename(2)，"pending" 目录允许使用 mkdir(2) 和 rmdir(2)。如果在 "pending" 目录中创建了一个 item，它的属性可以随意修改，用户空间通过将 item 重命名到 "live" 目录中来提交，此时，子系统接收到 ->commit_item() 回调。如果所有所需的属性都被填充，该方法返回0，item 被移到 "live" 目录下。

由于 rmdir(2) 在 "live" 目录中不起作用，所以必须关闭一个 item，或者说使其 "uncommitted"，同样，这也是通过 rename(2) 来完成的，这次是从 "live" 目录回到 "uncommitted" 目录，并通过 ct_group_ops->uncommit_object() 方法通知子系统。



> 参考：https://www.kernel.org/doc/Documentation/filesystems/configfs/configfs.txt
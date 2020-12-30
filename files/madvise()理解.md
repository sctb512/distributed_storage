# madvise()理解

> madvise = memory advise，表示对内存使用的建议。为什么需要这个函数呢？系统内核中使用内存策略已经固定（例如：分页，缺页 etc. ），用户程序无法直接干预。在某些情况下，例如，播放一部电影，假设系统分页为每页4kB，那么，播放这部电影就会不断发生缺页中断，而系统处理缺页中断的开销非常大的。这时，就可以使用madvise函数给内核一个建议，让它提前按顺序将电影内容加载进内存，从而避免发生缺页中断。

## 1. 函数介绍

使用madvise()需要包含如下头文件：

```c
#include <sys/mman.h>
```

其函数声明如下：

```c
int madvise(void *addr, size_t length, int advice);
```

函数返回值：函数执行成功时，madvise（）返回零。 错误时，返回-1并设置了errno。

在linux环境下，可以在以下路径找到该头文件：

```c
vim /usr/include/x86_64-linux-gnu/sys/mman.h
```

内容如下：

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201024101935.png" style="zoom:50%;" />

madvise() 函数有3个参数，\_\_addr是要提供建议内存的起始地址，\_\_len是从起使地址开始的内存长度，\_\_advice是建议的类型。在大多数情况下，用户程序通过madvise（）提供这些建议的目的是提高系统或应用程序的性能。
最开始，madvise（）函数仅支持一组 “常规” 的建议，这些建议值也可以在其他几种 madvise() 的实现中使用。 随后，又增加了许多“Linux特有”的建议。



## 2. "常规"的建议

下面列出的建议值使应用程序能够告诉内核它希望如何使用某些映射或共享的内存区域，以便内核可以选择适当的预读和缓存技术。 这些建议值不影响应用程序的语义（MADV_DONTNEED除外），但可能会影响其性能。 除MADV_DONTNEED外，下面列出的所有建议值在POSIX指定的posix_madvise（3） 函数中都有类似值，并且这些值具有相同的含义。

- MADV_NORMAL    

  默认值，无含义。

- MADV_RANDOM  
  
    应用程序对该内存区域中页面的引用是随机的。（因此，预取操作可能没有以前那么有用。）
  
- MADV_SEQUENTIAL
  
    应用程序对该内存区域中页面的引用是按顺序的。（因此，对给定的内存范围可以大胆地预取，并且，在访问后不久，可以将其释放。）
    
- MADV_WILLNEED
  
        应用程序将在不久后再次访问该内存区域中页面。 （因此，预取有一些页面是不错的想法。）
        
- MADV_DONTNEED
  
        应用程序在将来不会再访问这些内存区域中的页面。 （应用程序不再使用该内存区域，因此内核可以释放与其关联的资源。）
        
        成功执行MADV_DONTNEED操作后，将更改指定区域中的内存访问语义：对该范围内页面的后续访问将成功，但会导致要么从该基本映射文件的最新内容中重新填充内存内容 （用于共享文件映射，共享匿名映射和基于shmem的技术，例如System V共享内存段），要么按需填零页面（用于匿名专用映射）。
        
        请注意，将MADV_DONTNEED应用于共享映射时，可能不会立即释放该范围内的页面。 内核可以在合适的时间释放页面， 但是，调用程序的可用内存集大小（RSS）将立即减小。
        
        MADV_DONTNEED无法应用于锁定的页面、巨大的TLB页面或VM_PFNMAP页面。 （标记有内核内部VM_PFNMAP 标志的页面是不由虚拟内存子系统管理的特殊内存区域，这类页面通常由将页面映射到用户空间的设备驱动程序创建。）



## 3. "linux特有"的建议

以下特定于Linux的建议值在POSIX指定的posix_madvise（3） 中没有对应项，并且在实现上可用的madvise（）接口中可能有也可能没有对应项。 请注意，其中一些操作会更改内存访问的语义。

- MADV_REMOVE (since Linux 2.6.16)

  释放给定范围的页面及其关联的后备存储。 这等效于在后备存储的相应字节范围内打孔（请参见fallocate（2））。 对指定地址范围内的后续访问将看到只包含零的字节。

  指定的地址范围必须映射为共享且可写。 此标志不能应用于锁定的页面、巨大的TLB页面或VM_PFNMAP页面。

  在最初的实现中，仅tmpfs（5）支持MADV_REMOVE；但从Linux 3.5开始，任何支持fallocate（2）FALLOC_FL_PUNCH_HOLE 模式的文件系统也都支持MADV_REMOVE。 Hugetlbfs失败时，错误为EINVAL，其他文件系统失败，错误为EOPNOTSUPP。

- MADV_DONTFORK (since Linux 2.6.16)

  在fork（2）之后，子进程不可以使用该范围内的页面。 如果父级在fork（2）之后写入页面，该建议能有效防止 copy-on-write 语义更改页面的物理位置。 （这样的页面位置改变会给进入页面的DMA硬件带来问题。）

- MADV_DOFORK (since Linux 2.6.16)

  撤销 MADV_DONTFORK 的效果， 恢复默认行为， 即一个映射会在 fork(2) 之间继承。

- MADV_HWPOISON (since Linux 2.6.32)

  在addr和length指定的范围内清理页面，并像处理硬件内存损坏一样处理后续对这些页面的引用。 这个操作只对有特权(CAP_SYS_ADMIN)的进程有效。 此操作可能导致调用进程收到一个SIGBUS，页面被取消映射。
  这个功能是为了测试内存错误处理代码；只有当内核被配置为CONFIG_MEMORY_FAILURE时，这个功能才可用。

- MADV_MERGEABLE (since Linux 2.6.32)

  为addr和length指定范围内的页面启用 Kernel Samepage Merging (KSM)。 内核会定期扫描那些被标记为可合并的用户内存区域，寻找具有相同内容的页面。 这些页面会被一个受写保护的页面所取代（如果以后有进程想要更新页面的内容，那么这个页面会自动被复制）。 KSM只合并私有的匿名页面（参见mmap（））。

  KSM功能适用于生成许多相同数据实例的应用程序（例如KVM等虚拟化系统）。 它可能会消耗大量的处理能力，请谨慎使用。 更多细节请参考Linux内核源文件 <font color="green">Documentation/admin-guide/mm/ksm.rst</font>。

  MADV_MERGEABLE 和 MADV_UNMERGEABLE 操作只有在使用CONFIG_KSM配置内核的情况下才能使用。

- MADV_UNMERGEABLE (since Linux 2.6.32)

  撤消之前的MADV_MERGEABLE操作对指定地址范围的影响；KSM在addr和length指定的地址范围内撤消它所合并的任何页面。

- MADV_SOFT_OFFLINE (since Linux 2.6.33)

  使由addr和length指定范围内的页面软离线。 指定范围内每个页面内存仍然被保留，但原始页面已经离线。（即，下次访问时，相同的内容可见，但是在新的物理页框中。原始页面将不再使用，并从正常的内存管理中删除。） MADV_SOFT_OFFLINE操作的效果对于调用过程是不可见的（即，不改变其语义）。
  此功能旨在测试内存错误处理代码。 仅当内核配置有CONFIG_MEMORY_FAILURE时，此选项才可用。

- MADV_HUGEPAGE (since Linux 2.6.38)

  为addr和length指定范围内的页面启用透明大页(THP)。 目前，透明大页只适用于私有匿名页（参见 mmap（2））。内核会定期扫描被标记为大页候选区域，用大页替换它们。 当区域自然对齐到大页的大小时，内核也会直接分配大页（见posix_memalign（2））。
  这个特性主要是针对那些使用大块数据映射，并且一次访问该内存的大块区域（如QEMU等虚拟化系统）。 这很容易造成内存浪费（例如，一个仅访问1个字节的2 MB映射将导致2 MB的内存加载，而不是一个4 KB页面）。 更多细节请参考Linux内核源文件 <font color="green">Documentation/admin-guide/mm/transhuge.rst</font>。

  大多数常见的内核配置都默认提供了和MADV_HUGEPAGE类似的行为，因此MADV_HUGEPAGE通常是不必要的。 它主要适用于嵌入式系统，在这些系统中，MADV_HUGEPAGE风格的行为可能在内核中没有被默认启用。 在这样的系统中，这个标志可以用来选择性地启用THP。 无论何时使用MADV_HUGEPAGE，它总是在内存区域，其访问模式应该是开发者事先知道，当透明大页被启用时，不会增加应用程序的内存占用。

  MADV_HUGEPAGE 和 MADV_NOHUGEPAGE 操作只有在内核配置了CONFIG_TRANSPARENT_HUGEPAGE的情况下才能使用。

- MADV_NOHUGEPAGE (自Linux 2.6.38起)

  确保addr和length指定的地址范围内的内存不会被透明大页（THP）支持。

- MADV_DONTDUMP (自 Linux 3.4 起)

  从核心转储中排除addr和length指定范围内的页面。 这对于有大面积内存的应用程序来说是很有用的，因为这些内存在核心转储中是没用的。 MADV_DONTDUMP的效果优先于通过 <font color="green">/proc/[pid]/coredump_filter </font>文件设置的位掩码（见core(5)）。

- MADV_DODUMP (自 Linux 3.4 起)

  撤销之前 MADV_DONTDUMP 的效果。

- MADV_FREE (自 Linux 4.5 起)

  应用程序不再需要addr和len指定范围内的页面。 内核可以释放这些页面，但是释放可能会被延迟，直到出现内存压力。 对于每一个被标记为要释放但尚未释放的页面，如果调用者向该页面写入内容，释放操作将被取消。 在一次成功的MADV_FREE操作后，当内核释放页面时，任何陈旧的数据（即脏的、未写入的页面）将被丢失。然而，后续对该范围内的页面的写入将成功，然后内核不能释放这些脏污的页面，这样调用者可以始终看到刚刚写入的数据。 如果没有后续的写入，内核可以在任何时候释放这些页面。一旦范围内的页面被释放，调用者在后续的页面引用时将看到零填充的需求页面。

  MADV_FREE操作只能应用于私有的匿名页面（参见mmap(2)， 在4.12版本之前的Linux中。

  当在无交换系统中释放页面时，给定范围内的页面会被立即释放，而不考虑内存压力。

- MADV_WIPEONFORK (自Linux 4.14起)

  fork（2）之后，在这个范围内给子进程提供零填充的内存。 这在fork服务器中很有用，以确保敏感的每进程数据（例如PRNG种子、加密秘密等）不会交给子进程。
  MADV_WIPEONFORK 操作只能应用于私有匿名页面（参见 mmap(2)）。
  在由 fork(2) 创建的子进程中，MADV_WIPEONFORK 设置在指定的地址范围内保持不变。 这个设置在execve(2)期间被清除。

- MADV_KEEPONFORK (自 Linux 4.14 起)

  撤销之前MADV_WIPEONFORK的效果。



## 4. madvise(MADV_DONTNEED)和free()

在c语言开发中，很多时候都会用到free()函数释放已经分配的内存，但调用free()函数释放指定地址的内存后，该地址将被收回，放在空闲内存列表中，如果此时再访问该被释放的地址，将会产生段错误。

有时候，我们需要在保留地址的情况下，将某内存地址的内容释放掉，这时就可以使用 madvise(addr，len，MADV_DONTNEED) 来实现。madvise(addr，len，MADV_DONTNEED)  函数执行后，addr对应的内存空间将会被释放，但该地址不会被操作系统收回而放在空闲内存列表中

此时，如果再次访问该地址，不会产生段错误（Segment fault），如果我们使用 userfaultfd（）注册了该地址，可以在用户态捕获该缺页中断，并使用自己的代码流程进行处理。

下面是一段测试代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include <sys/mman.h>
#include <unistd.h>

int main() {
        printf("++++ before malloc:\n");
        system("free -h");

        long len = 1024 * 1024 * 1024;          // 1 GB
        // char *buffer = (char *)malloc(sizeof(char) * len);
  			char *buffer;

        posix_memalign((void **)&buffer, sysconf(_SC_PAGESIZE), len);

        memset(buffer, 'M', len);
        printf("buffer[0]: %c\n", buffer[0]);

        printf("\n\n");
        printf("++++ after malloc:\n");
        system("free -h");

        int ret = madvise(buffer, len, MADV_DONTNEED);
        if (ret == -1) {
                printf("madvise ret: %d\n", ret);
                printf("err: %s\n", strerror(errno));
        }

        printf("\n\n");
        printf("++++ after madvise:\n");
        system("free -h");

        printf("buffer[0]: %d\n", buffer[0]);
}
```

首先，我们使用posix_memalign（）函数分配了1G的内存空间，然后使用memset（）确保分配的地址在内存中都分配了空间，最后调用 madvise(buffer, len, MADV_DONTNEED) 建议内核释放该地址空间。因为该buffer没有使用mmap（）映射文件，也没有通过userfaultfd（）注册缺页中断，所以，再次访问该地址范围时，将会得到 0 值。

结果如下：

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201025170931.png" style="zoom:50%;" />

注意：posix_memalign((void **)&buffer, sysconf(_SC_PAGESIZE), len) 分配的空间是页对齐的，madvise（）只能传入页对齐的地址，如果传入malloc（）分配的地址，将会报错：<font color="red">Invalid argument</font>


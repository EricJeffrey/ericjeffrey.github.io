
# 内核内存管理模块入门与介绍

大致介绍下内存管理的一些大框架，帮助新人快速入门。

## 前置知识(非严谨描述)

### 虚拟地址与物理地址(虚拟内存/物理内存)

用户态看到的地址如`char *addr=malloc(16);`中的`addr`是虚拟地址(进程可用地址范围内中一个数值)，它实际对应到内存条中的物理地址可能是某个`paddr(physical address)`(由内核决定，用户态一般无法控制)。

内核通过多级页表的方式实现虚拟地址到物理地址的映射关系`addr->paddr`，并将这个多级页表配置到CPU中。对于CPU来说，它执行指令时拿到的值也是虚拟地址。它会通过内核创建的多级页表，将addr翻译为物理地址paddr，然后从内存条中读取实际的值。

### pagefault

`addr->paddr`的映射关系是可以动态创建的，典型方式是mmap之后通过pagefault去创建。

如果CPU在读取虚拟地址addr的时候，发现内核配置的页表上找不到有效的paddr，就会触发一次访存失败(data/instruction abort)，然后进入内核中执行pagefault的处理流程。

pagefault处理流程负责分配物理页(内核是按照页的粒度管理内存，不是byte)，并创建相应的映射关系。之后就可以让CPU继续执行对应访存指令访问对应地址的内容了。

因此对于进程来说，内存分配可以分为**虚拟内存创建**与**映射建立两部分**。

## 系统调用

系统调用是内核提供的直接入口，根据他们能大致了解内存管理的目标 -> 即向上层应用提供分配内存、释放内存等能力。

建议每一个都去查查man手册如`man madvise`，或者官方mannual网站上查看，如[madvise(2)-man7.org](https://www.man7.org/linux/man-pages/man2/madvise.2.html)

下面简要介绍:

- mmap - 用户态进行内存分配的主要函数，提供了多种多样的内存分配能力，包括映射文件的内容到内存中(以下简称文件映射)，分配普通匿名内存(即所谓的malloc出来的堆内存)，分配虚存同时建立映射等能力。
	- *注: mmap里面的map起初表示将文件页映射到内存中，但是这个接口也包含不涉及文件的操作(MAP_ANONYMOUS)*
- munmap - 释放一段由mmap映射出来的内存(包含虚存与物存)
- mremap - 将一段虚存重新map到另一段，或者扩展/缩小某段虚存
- pagefault - 处理CPU的data/instruction abort
- fork/clone - 创建子进程/线程的方式，fork要拷贝整个进程的虚存到子进程，clone包含一些与地址空间相关的flag
- mprotect - 修改一段内存的读/写/执行属性
- madvise - 提供某段内存的'advise'给内核，例如`MADV_WILLNEED(内存后续会被使用)`,`MADV_DONTNEED(后续不会被使用)`,`MADV_REMOVE可以移除相关的映射`等等
- mlock/mlock2/mlockall - 将某段虚存lock到内存中，主要用来防止内存被换出(swap out)。mlock2和mlockall提供了一些额外的flag控制lock动作发送的时机
- munlock/munlockall - 上述操作的逆操作
- mincore - 检查一段虚存是否有对应的物存
- msync - 用于文件映射的虚存上，将内存写入的值flush到文件上
- shmat/shmdt/shmctl/shmget - System V标准定义的共享内存接口，用来创建不同进程之间共享的内存
- shm_open/shm_unlink - POSIX标准定义的共享内存接口
- swapoff/swapon - 开启/关闭swap
	- swap是一种系统低内存时回收内存的功能
- procfs
  - /proc/sys/vm/... - 包含许多内存相关的配置
  - /proc/pid/maps, /proc/pid/smaps - 可以观察进程中不同虚存段的各种属性
  - /proc/pid/oom_xxx - OutOfMemory的一些配置
  - /proc/pid/stat, statm, status - 进程的状态统计信息
  - /proc/pid/pagemap - 提供了观察虚存对应的物存映射的信息
  - /proc/pid/mem - 提供了访问任何进程的虚存的手段
  - /proc/meminfo, /proc/memview - 提供了全局的内存相关的统计信息
  - ...
- cgroup/memory/... - CGroup中的内存相关的group
- brk/sbrk - 用于调整进程数据段的边界，从而达到分配或者释放内存的能力

## 模块介绍

- 物理内存管理(This is Kernel!)
- 虚拟地址空间管理
- 映射管理
- Low Memory! 回收
	- LRU
	- Swap
- 迁移Migrate
	- 连续物理内存CMA
	- 碎片整理
...

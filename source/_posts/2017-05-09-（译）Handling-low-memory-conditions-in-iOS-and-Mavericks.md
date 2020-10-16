---
title: （译）Handling low memory conditions in iOS and Mavericks
categories: 翻译
abbrlink: 50ed72f
date: 2017-05-09 00:00:00
---

> 翻译自 Jonathan Levin’s site   原文链接：http://newosxbook.com/articles/MemoryPressure.html

## About

OS X 和 iOS 中的内存压力是虚拟内存管理的一个非常重要的方面，在我的书中[[1\]](https://amywushu.github.io/2017/05/09/译-Handling-low-memory-conditions-in-iOS-and-Mavericks.html#references)已经有了一些介绍。 我提到的 Jetsam/memorystatus 机制，已经随着时间的推移发生了重大变化，最终形成了最近在 Mavericks 中引入的一些非常重要的系统机制和系统调用。 在使用我的 OS X 和 iOS 的Process Explorer 时，我遇到了这些新增加的问题，因此在这里记录。 这是作为本书第12章的补充，当然也可以单独阅读。

<!-- more -->

### Why should you care? (Target Audience)

物理内存（RAM），是 CPU 的另一个方面，也是系统中最稀少的资源，是最有可能导致竞争的资源，因此 apps 争夺每一个有效的 bit。应用程序的内存与性能直接相关 - 通常是以别人的为代价。 在 iOS 中，没有能够交换内存的交换空间，这使得内存资源更为重要。 本文旨在让您下次调用 malloc() 或 mmap() 之前再三考虑，以及阐明 iOS 上最常见的崩溃原因 -低系统内存的原因。

## Prerequisite: Virtual Memory in a nutshell

无论一个应用程序是如何编程的，它都必须在内存空间中运行。 这个空间是一个应用程序可以控制自己的代码，数据和状态的地方。 当然，如果这样一个空间与其他应用程序隔离，这是非常有益的，因为提供了更好的安全性和稳定性。 我们将这个空间称为应用程序的虚拟内存，它是应用程序的一个定义特性之一：应用程序的所有线程将共享相同的虚拟内存空间，也因此被定义为处于同一进程。

虚拟内存中的术语“虚拟”意味着存储器空间虽然与所讨论的进程有很大关系，但并不完全对应于系统里的实际内存。 这表现在几个方面：

- 虚拟内存空间可以超过可用的实际内存量 - 取决于所讨论的处理器字大小和操作系统，虚拟内存空间可高达4GB（32位）或256TB（64位）[[1\]](https://amywushu.github.io/2017/05/09/译-Handling-low-memory-conditions-in-iOS-and-Mavericks.html#footnotes)。这，尤其在后一种情况下，可以远远超出现有的可用内存量。
- 实际上，虚拟内存并不存在：给出如此巨大的超过物理内存支持能力的内存空间，只有当应用程序明确请求内存（即分配），系统才会为支持的虚拟内存映射物理内存。因此，一个进程的虚拟内存的影像是非常稀疏的，在内存中就像在一片浩瀚的虚空海洋中的“孤岛”。
- 即使分配了，虚拟内存可能仍然是虚拟的： - 当你调用malloc(3) 时，并不意味着系统会跳转并找到相应数量的RAM来实质地分配你的内存。大多数情况下，程序员的分配远远超过他们所需要的。因此，malloc(3) 操作，只分配页表入口（entries），很少分配内存本身。实际上，当访问内存时（比如说 memset(3)），才会导致物理分配。
- 系统可以备份内存到磁盘或网络上 - 也称为“交换”内存到后台存储。 OS X 一般使用交换文件（在/var/vm中）。iOS没有交换机制。
- 您使用的虚拟内存可能共享或不共享 - 操作系统保留使用与其他进程隐式共享你的虚拟内存的权限。这适用于使用文件备份的内存（即通过调用 mmap(2) 声明的内存）。如果你的进程和另一个进程 mmap(2) 相同的文件，操作系统可以给你们每个进程一份你的私有虚拟副本，但实际上是同一个物理副本。上述物理副本将被标记为不可写。只要每个进程都只是从内存中读取，那么一份就足够了。然而，如果任何人向这样的隐式共享内存写入，写入过程将触发页面错误，这将导致内核执行一个 Copy-On-Write 机制（COW），从而产生一个可以修改内容的新物理副本。

把上面总结一下，我们可以得到以下“公式”：

```
  VSS = RSS + LSS + SwSS  
各项含义：   
VSS　　虚拟大小，由top，ps(1) 和其他的命令可以看到   
RSS　　驻留大小 - 进程的实际RAM占用空间。 也可以在 top(1)，ps(1) 等中显示   
LSS 　　“Lazy” 大小 - 系统同意分配但尚未分配的内存   
SwSS　　“交换”大小 - 以前在RAM中但被推出交换的内存。在 iOS 中，一直为0　　　　
```

以上所有的东西可以通过一个简单的例子清楚地展示 - 在任一进程中使用 vmmap(1)，这里以 shell 本身为例：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片1.png)

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片2.png)

**术语**
在本文中，使用以下术语：

- Page - 内存管理的基本单元。在 Intel 和 ARM 中，通常为4k（4096），在 ARM64 中通常为16K。您可以使用 OS X 上的 pagesize(1) 命令（或任何一个操作系统上的 sysctl hw.pagesize）来确定默认页面大小是多少。英特尔架构支持超级页面（8k）和巨大页面（2MB），但在实践中，这些页面相对较少。
- Phsyical Memory/RAM - 安装在主机（Mac 或 i-Device）上的有限数量的内存。你可以使用 `hostinfo(1)` 命令获取这个值。
- Virtual Memory - 由程序或系统本身分配的内存，通常是通过调用 `malloc(3)，mmap(2)` 或更高级别的调用（例如Objective-C 的 [alloc]等）分配。虚拟内存可能是私有的（由单个进程所有）或共享的（由2+进程所有）。共享内存可以是明确地或隐式地共享。
- Page Fault - 当内存管理单元（MMU）检测到违规访问虚拟内存时，即发生以下情况之一：
  - 访问未分配的内存：取值一个之前未被分配的内存指针 - XNU将其转换为一个 EXC_BAD_ACCESS 异常，并且进程将接收到分段错误（SIGSEGV，Signal#11）。
  - 访问已分配但未提交的内存：取值一个先前已分配但尚未使用的内存指针（或相应的madvise(2)） - XNU拦截，并意识到它不能再等待，必须分配物理页。当分配页面时，将冻结导致故障的线程。
  - 访问内存但不符合其权限：内存页面以与标准 UNIX 文件权限相似的方式受 r/w/x 保护。尝试写入只读目标（r–或rx）将导致页面错误，XNU 将其转换为总线故障（SIGBUS，Signal#7）或强制执行 Copy-On-Write（COW）操作（如果是隐式共享的）。

**工具**
苹果提供了几个重要的工具来检查虚拟内存：

- vmmap(1) - 检查单个进程的虚拟内存，以类似于Linux 以 /proc/<pid>/maps 的方式布局其 “map”。
- vm_stat(1) - 从系统范围的角度提供有关虚拟内存的统计信息。 这实际上只是一个调用 Mach host_statistics64 API 的封装器，打印出 vm_statistics64_t（来自<mach/vm_statistics.h>）。
- top(1) - 提供与性能有关的全系统以及每个进程的统计信息。 在其中，MemRegions，PhysMem 和 VM 统计信息涉及虚拟内存。
  我厚脸皮地在这里推广一下自己的工具，process explorer （procexp），它提供了（IMHO）比 top(1) 更好的功能（包括更丰富的内存统计）。

## Memory Pressure

Mach 层内部有两个计数器定义内存压力：

- vm_page_free_count：目前有多少页 RAM 是空闲的
- vm_page_free_target：至少有多少页 RAM 应该被释放。
  你可以使用 sysctl 轻松查看这些：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片3.png)

如果空闲页面的数量低于目标数量 - 也就是有内存压力的情况（当然还有其他潜在的情况，但为了简化起见我省略了这些 [[2\]](https://amywushu.github.io/2017/05/09/译-Handling-low-memory-conditions-in-iOS-and-Mavericks.html#footnotes)）。你也可以使用 sysctl(8) 来查询 `vm.memory_pressure` 的值。在 OS X 10.9 及更高版本中，你也可以查询 `kern.memorystatus_vm_pressure_level`，其值为 1（NORMAL），2（WARN）或 4（CRITICAL）

在内核初始化之后，主线程变为 vm_pageout，并产生一个专用的线程，并被贴切地叫做 vm_pressure_thread，用来监测压力事件。这个线程是空转的（阻塞自己）。当检测到压力时，线程将被 vm_pageout 唤醒。这种行为已在 XNU 2422/3（OSX 10.9/iOS 7）中修改（最显著的是转移到了 vm_pressure_response 中封装）。

只有当 VM_PRESSURE_EVENTS 是 #define 的时候，压力处理才会被编译到 XNU 中。如果不是（比如，自定义编译），vm_pressure_thread 在2050 版本中什么都不做，甚至在 2422/3 中也不会开始。此外，在iOS内核中，定义 CONFIG_JETSAM 会更频繁地将内存处理调度到 memorystatus 线程以及更新其计数器（稍后会讲）来更改某些行为。

### [mach]_vm_pressure_monitor

XNU 导出未记录的系统调用#296，vm_pressure_monitor（bsd/vm/vm_unix.c），它是 mach_vm_pressure_monitor（osfmk/vm/vm_pageout.c）以上的一个封装器。系统调用（以及相应的内部 Mach 调用）定义如下：

> int vm_pressure_monitor（int wait_for_pressure，int nsecs_monitored，uint32_t * pages_reclaimed）;

这个调用将立即返回或者阻塞（当 `wait_for_pressure` 不为零时）。它将返回 page_reclaimed 在 nsecs_monitored 的计数中释放了多少个物理页面（并不是真的循环迭代那么多的 nsecs）。其返回值表示需要提供多少页面（上面的sysctl(8) 输出中的 `vm.page_free_wanted`）。调用系统调用很简单，而且不需要 root 权限。 （再次重复，您也可以使用sysctl(8) 来查询 `vm.memory_pressure`，尽管这个操作不会等待内存压力）。

您可以使用 “vmmon” 参数运行 process explorer 来尝试这个系统调用（否则，process explorer 将在交互模式下在单独的线程中执行此操作，以显示压力警告）。指定 “oneshot” 的附加参数将会在不等待压力的情况下直接调用。否则，调用将等待，直到检测到压力：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片4.png)

但是系统实际上是如何收回内存的？对于此，需要涉及到 memorystatus。

## MemoryStatus and Jetsam

当 XNU 移植到 iOS 时，苹果遇到了移动设备限制引起的重大挑战 - 没有交换空间。与桌面相比，虚拟内存可以“溢出”到外部存储器，在这里并不适用（主要是由于闪存的限制）。因此，内存已经成为一个更重要（更稀缺）的资源。

引入：MemoryStatus。最初在 iOS 中引入的这种机制是一个内核线程，负责处理低 RAM 事件，以 iOS 认为可能的唯一方式：丢弃（弹出）尽可能多的RAM，为应用程序释放内存 - 即使当这种方式意味着杀死应用程序。这就是 iOS 的 jetsam 机制，可以在 XNU 源代码中看到 `#if CONFIG_JETSAM` 编译选项。在 OS X 中，memorystatus 代表的不是 kill，而是那些标记为空闲退出的进程，这是一种更温和的方式，更适合于桌面环境[[3\]](https://amywushu.github.io/2017/05/09/译-Handling-low-memory-conditions-in-iOS-and-Mavericks.html#footnotes)。 使用 dmesg，以及 grep 可以看到 memorystatus 的操作：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片5.png)

memorystatus 线程是一个单独的线程（也就是不直接与 vm_pressure_thread 相关），它在XNU的 BSD 部分启动（通过在 `bsd/kern/bsd_init.c` 中调用 `memorystatus_init`）。如果定义了 CONFIG_JETSAM（iOS），memorystatus 会启动另一个线程 `memorystatus_jetsam_thread`，它大部分时间在阻塞循环中运行，当memorystatus_available_pages <= memorystatus_available_pages_critical 时被唤醒，杀死处于 memory list 中最上方的进程，然后再次阻塞。

在 iOS 中，memorystatus/jetsam 不会打印消息，但肯定会在 `/Library/Logs/CrashReporter/LowMemory-YYYY-MM-DD-hhmmss.plist` 中留下其杀死程序的痕迹 - 这些日志由 CrashReporter 生成，类似于包含了 dump 的崩溃日志。如果你有越狱设备，一个简单的方法可以强制 jetsam 大规模执行，运行一个小的二进制文件，不停的分配和 memset() 大小为8MB 的内存块（这个留给感兴趣的读者练习），然后运行它。你将会看到应用程序死亡，直到这个犯规的二进制文件（最终）被杀死。日志将看起来像这样：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片6.png)

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片7.png)

（请注意，您可以在非越狱设备上执行此操作，如果您已将其配置为开发用，您可以在 Objective-C 中创建一个简单的 iOS 应用程序，并执行相同的分配，然后通过 XCode 的 Organizer 收集日志） 。

应该注意的是，Jetsam 无情地彻底杀死一个进程，并非罕见：Linux（以及它的继承者，Android）在 “OOM”（out-of-memory）killer 中有一个类似的机制，它持有了每个进程的（可能是可调整的）分数，当遇到内存不足时，杀死高分数进程。在桌面 Linux 中，OOM 在系统交换空间耗尽时唤醒；在 Android 中，要更早一点，当 RAM 运行内存较低时就被唤醒。Android 的方法是得分驱动（该分数实际上是一种启发式方法，取决于使用了多少 RAM，以及怎样的频率），而 iOS 的方法是基于优先级的。

从 XNU 2423 开始，Jetsam 使用了一种 “priority bands”（参见 <sys/kern_memorystatus.h> JETSAM_PRIORITY 常量），也就是说 jetsam 跟踪的进程被维护在内核空间（memstat_bucket）的21个链表的数组中。Jetsam 将选择最低优先级元（以0或 JETSAM_PRIORITY_IDLE 开头）的第一个进程，如果当前优先级为空（参考 memorystatus_get_first_proc_locked，在bsd/kern/kern_memorystatus.c 中），则顺移至下一个优先级列表。进程的默认优先级为18，允许 jetsam 选择空闲和后台进程，这些进程的顺序排在交互式和可能重要的进程之前。如下图所示：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片8.png)

Jetsam 还有另一种操作方式，它设置进程内存的 “high water mark”，并且将彻底杀死超过其 HWM 的进程。 当一个任务的 RSS 内存超过系统范围限制时，Jetsam 中的 HWM 模式就会被触发（更准确地说，是任务的 phys_footprint 内存，其中包括 RSS，也包括 compressed 和 I/O Kit 相关的内存）。 HWM 可以通过 memorystatus_control 操作 #5（MEMORYSTATUS_CMD_SET_JETSAM_HIGH_WATER_MARK，稍后讨论）来设置 。

在 iOS 上，Launchd 可以设置 jetsam 的优先级。 之前这个操作是在每个守护进程的基础文件（也就是在它的 plist 文件）中完成的。 现在看来，这些设置已被移至 com.apple.jetsamproperties.*model*.plist（例如 N51(5s)，J71(iPad Air)等）。 如下：

```
<dict>
        <key>CachedDefaults</key>
    <!-- Array of dict entries, with key being daemon name e.g. -->
        <dict>

                <key>com.apple.usb.networking.addNetworkInterface</key> 
                <dict>
                        <key>JetsamMemoryLimit</key>
                        <integer>integer>6</integer>
                        <key>JetsamPriority</key> 
                        <integer>integer>3</integer>
                        <key>WellBehaved</key> 
                        <true/>
                </dict>
..
```

由于 RAM 消耗而彻底杀死一个进程可能看起来过于苛刻，但对于缺乏交换机制的系统来说，实际上能做的非常少。在 Jetsam 杀死一个进程之前，memorystatus 允许进程“挽回自身”，避免不合适的终止，可以通过获取 memorystatus 线程首先向作为终止的“候选”进程发送一个 kernel note（又称 kevent） 。这个 knote（`NOTE_VM_PRESSURE，<sys/event.h>`）将被 EVFILT_VM kevent() 过滤器拾取，就像 UIKit 把它转换为 `didReceieveMemoryWarning` 通知一样，这无疑是 iOS App 开发人员所熟悉的（也被其讨厌）。Darwin 的 libC 和 GCD 也加入了内存压力处理程序，具体如下：

- Darwin 的 LibC（<malloc/malloc.h>）定义了一个 `malloc_zone_pressure_relief`（从 OSX 10.7/iOS 4.3 开始）
- LibCache（<cache.h>）定义了缓存成本（对于 cache_set_and_retain），允许发生压力事件时自动清除缓存。
- GCD（<dispatch/source.h>）定义了 `DISPATCH_SOURCE_TYPE_MEMORYPRESSURE`（从OSX 10.9 起）

一般来说，注册了内存压力的应用程序（直接通过 Darwin API 或间接通过 UIKit）应该减少其缓存和潜在的不需要的内存（应该注意的是，遍历内存结构可能导致页面错误，会加剧内存压力）。 UIKit 是不开源的，但是当 UIApplication 遇到内存警告时，jtool 提供了一个很好的反汇编来演示它的行为：

![](/images/2017-05-09-译-Handling-low-memory-conditions-in-iOS-and-Mavericks/图片9.png)

然而有时候，释放内存可能不足以缓解内存压力。 大多数情况下，释放的内存可能很快被另一个不愿意释放它的应用程序占用。 在这些情况下，最后的手段是杀死潜在候选人列表中最上方的进程 - 所以 Jetsam 出现了。

**Controlling memorystatus**

一个线程可以随意地决定杀死进程可能有点危险。因此，苹果使用几种API来“统治”Jetsam/memorystatus。当然，这些都是私有的和无文档记录的（如果在应用程序中使用它们，苹果可能会 kill 你的开发人员帐户），它们是：

- 使用 sysctl kern.memorystatus_jetsam_change：可以从用户空间更改 Jetsam 的优先级列表。这有点像 Linux 的 oom_adj，它允许进程通过指定负调整数（有效地降低它们的分数）来逃避 OOM 的惩罚。同样，在 iOS 中，launchd（启动所有应用程序的进程）可以设置 Jetsam 优先级列表。 （例如，参考 com.apple.voiced.plist，它指定 JetSamMemoryLimit(8000) 和 JetsamPriority(-49) ），sysctl 内部调用memorystatus_list_change（在 bsd/kern/kern_memorystatus.c 中），它会再次设置优先级和状态标志（active，foreground 等）。 - 跟 Linux 类似，在这种情况下，Android的处理机制是 “Low Memory Killer”（在运行时可以根据应用程序/活动的前台状态来调整 OOM_ADJ，因此更喜欢首先杀死后台应用程序）。这种方法可以运行到 iOS 6.x。
- 使用 memorystatus_control(#440) 系统调用：在 xnu 2107 某处（即早在 iOS 6 而不是直到 OS X 10.9）中介绍，这个（无文档记录的）系统调用能够通过使用几个“命令”之一，来控制 memorystatus 和 jetsam（后者在 iOS 上），如下表所示：

| **MEMORYSTATUS_CMD_ const**       | **availability**                        | **usage**                                                    |
| :-------------------------------- | :-------------------------------------- | :----------------------------------------------------------- |
| GET_PRIORITY_LIST (1)             | OS X 10.9, iOS 6+                       | Get priority list - array of memorystatus_priority_entry from <sys/kern_memorystatus.h> Example code can be seen [Here](http://newosxbook.com/src.jl?tree=listings&file=mslist.c) |
| SET_PRIORITY_PROPERTIES (2)       | iOS only (or CONFIG_JETSAM)             | Update properties for a given proess                         |
| GET_JETSAM_SNAPSHOT (3)           | iOS only (or CONFIG_JETSAM)             | Get Jetsam snapshot - array of memorystatus_jetsam_snapshot_t entries (from <sys/kern_memorystatus.h> |
| GET_PRESSURE_STATUS (4)           | iOS (or CONFIG_JETSAM)                  | Privileged call: returns 1 if memorystatus_vm_pressure_level is not normal |
| SET_JETSAM_HIGH_WATER_MARK (5)    | iOS (or CONFIG_JETSAM)                  | Sets the maximum memory utilization for a given PID, after which it may be killed. Used by launchd for processes with a memory limit） |
| SET_JETSAM_TASK_LIMIT (6)         | iOS 8 (or CONFIG_JETSAM)                | Sets the maximum memory utilization for a given PID, after which it **will** be killed. Used by launchd for processes with a memory limit |
| SET_MEMLIMIT_PROPERTIES (7)       | iOS 9 (or CONFIG_JETSAM)                | Sets memory limits + attributes                              |
| GET_MEMLIMIT_PROPERTIES (8)       | iOS 9 (or CONFIG_JETSAM)                | Retrieves memory limits + attributes                         |
| PRIVILEGED_LISTENER_ENABLE (9)    | Xnu-3247 (10.11, iOS 9)                 | Registers self to receive memory notifications               |
| PRIVILEGED_LISTENER_DISABLE (10)  | Xnu-3247 (10.11, iOS 9)                 | Stops self receiving memory notifications                    |
| TEST_JETSAM (1000)                | CONFIG_JETSAM && (DEVELOPMENT 或 DEBUG) | Test Jetsam, kill specific processes (Debug/Development kernels only) |
| TEST_JETSAM_SORT (1001)           | iOS 9 && (DEVELOPMENT 或 DEBUG)         | Test Jetsam sorting (Debug/Development kernels only)         |
| SET_JETSAM_PANIC_BITS (1001/1002) | CONFIG_JETSAM && (DEVELOPMENT 或 DEBUG) | Alter Jetsam’s panic settings (Debug/Development kernels only) |

- 使用 posix_spawnattr_setjetsam：来自于 posix_spawnattr 系列的函数，但是没有被记录，仅存在于 iOS 上（这就是 iOS 7 上 launchd 如何处理 Jetsam 的）
- 使用 sysctl kern.memorypressure_manual_trigger 用于模拟内存压力水平，而不会实际占用内存 - 由 OS X 10.9 的 memory_pressure 实用工具(-S) 使用。 来自<sys/event.h >，NOTE_MEMORYSTATUS_PRESSURE_ [NORMAL | WARN | CRITICAL] 的值。

**Other memorystatus configurable values:**

- 使用 sysctl kern.memorystatus_purge_on_ * 的值（OS X）。这些值不会像 pageout 守护程序那样影响 memorystatus，而是强制它清除warning(2)，urgent(5) 或critica(8) 的值。 将这些值设置为0将清除禁用。
- 使用 memorystatus_get_level(#453)：该系统调用返回（到 int *）一个介于0到100之间的数字，指定可用内存的百分比。 只是诊断。使用在 Activity Monitor（和我的 Process Explorer）上，以显示 Mavericks 以及之后的版本的内存压力。

**Ledgers**

iOS 在 iOS 5（或5.1？）上重新引入了 ledgers，这个概念也被移植到了 OS X。 我说“重新引入”，是因为这是从开始的 Mach 设计以来， ledgers 已经存在，只是那时还没有真正实现。

ledgers 有助于解决资源利用率过多的问题。 与经典的 UN * X 模型不同（setrlimit(2)，被用户所熟知的 ulimit(1) ），ledgers 具有更细粒度的类似于 QoS 的模型，ledgers 为每个资源的每个时间单位分配一定的配额（RAM，CPU，I/O ），并且以奇怪的方式 “refills”。这允许操作系统提供 leaky-bucket 类型的 QoS 机制，并保证服务水平，如果一个进程超过它的 ledgers ，则会产生一个 Mach 异常（EXC_RESOURCE，#12，如果是内存服务的话）。

今后，苹果将完全转向基于 ledgers 的 RAM 管理机制，这是非常有意义的，尤其是在 iOS 这样一个资源稀缺（并且没有交换机制）的情况下。 Jetsam 将可能会被保留作为最后的手段。

## References:

1. [Mac OS X and iOS Internals, J Levin](http://newosxbook.com/articles/...)

## ChangeLog

- 3/1/2014 - Added jetsam properties plist from iPhone5s, and note about ledgers
- 2/10/2016 - Added jetsam/memorystatus commands for xnu 32xx (iOS 9, OS X 10.11). Also updated procexp to show mem limits on iOS

## Footnotes

1. 为了简单阐述，我们忽略了为某些给定进程提供的虚拟内存，实际上只是保留和映射以供内核使用的事实。顺便说一句，对于64位来说，是256TB，是由于硬件限制（加上没有人会真正使用它，更不用说是全64位的16EB）。 Mac OS X 以128-TB的47位（0x7fffffffffff）用做用户空间虚拟内存，最上面的（技术上为 0xffffffff8 …）128TB 为内核保留。
2. 再次声明，为了简化，我并没有说明实际的条件。
3. 我没有考虑空闲降级的过程，也就是说（10.9）进程可能被移动到空闲频段，以便它们成为空闲退出的候选者。 进程可以使用 PROC_INFO_CALL_DIRTYCONTROL 调用 proc_info 让内核跟踪其状态，当“dirty”和“clean”（空闲）自愿允许被杀死时，寻求保护以免被 kill 。与 vproc 机制（<vproc.h>）一起使用。
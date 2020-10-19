---
title: Hook 原理之 CydiaSubstrate（二）：MSHookMessageEx
categories: 源码解析
abbrlink: 9acae560
date: 2017-03-23 00:00:00
---

## 前情提要

在上一篇博文“[Hook 原理之 CydiaSubstrate（一）：MSHookMessageEx](https://harpersu00.github.io/92be8389.html)”中，我分析了 MSHookMessageEx 以前的老代码，也就是 iOS5 版本。那么在这一篇文中，我将通过逆向来分析新版的 MSHookMessageEx。

两个版本的大部分代码和流程都差不多，主要区别在于，当 hook 的方法不是本类方法的时候，老版本的代码是通过嵌入机器码来获取 IMP，新版本是通过构建 trampoline（蹦床）页来获取 IMP。

<!-- more -->

## 如何调试

我们使用 CydiaSubstrate 框架对 APP 作 hook 操作的话，它是会在 /Library/MobileSubstarte/DynamicLibraries/ 目录下生成一个 .dylib 文件和一个对应的 plist 文件。这个 .dylib 文件将在 App 启动时被 dyld 加载到程序中，对指定类的方法进行 hook。

1. 在 iPhone 的 /Library/MobileSubstarte/ 目录下有一个 **MobileSubstrate.dylib** 文件，这只是一个链接文件，它的真身是在 /Library/Frameworks/CydiaSubstrate.framework/Librarys/ 目录下的 SubstrateBootstrap.dylib 文件。
2. 这个动态库会被加载入每个 App 进程中，然后调用同目录下的 **SubstrateLoader.dylib** 动态库。
3. 通过 SubstrateLoader.dylib 来启动上级目录下的 Mach-O 文件： **CydiaSubstrate**。关于 MSHookMessageEx 的代码即在这个可执行文件中。
4. 最后通过 CydiaSubstrate 来加载该 App 对应的，位于 /Library/MobileSubstarte/DynamicLibraries/ 目录下的 hook dylib 文件，从而完成 hook 操作。

**这些动作都发生在 main 函数被调用之前。**

程序启动后，不会再调用 MSHookMessageEx 函数，而是直接通过已被设置的 IMP 值跳到自定义的 hook dylib 中。因此我们要进入 CydiaSubstrate 可执行文件中动态调试 MSHookMessageEx 函数，就需要**将断点下在第3步和第4步之间**。

准备工作：自己构建的 App，hook 该 App 的 dylib。（这两个程序均已在越狱 iPhone 上安装好）

首先添加 Symbolic Breakpoint（符号断点）`_objc_init`，然后运行 App，当程序停在断点处时，在控制台输入 `image list -o -f` 命令，可以看见，与 Cydia 相关的文件，目前就只加载了 MobileSubstrate.dylib。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/MobileSubstrate.png)

在 IDA 中打开它的真身 SubstrateBootstrap.dylib，可以看见自定义的函数只有一个 InitFunc_0，其余几个函数（exit、dlclose、dlopen、getenv）都是从外部引用的。而 InitFunc_0 函数也非常简单。主要是通过 dlopen 函数调用几个动态库，来作环境准备，其中我们要下断点的地方就在 dlopen 调用 SubstrateLoader.dylib 时：

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/SubstrateBootstrap_dlopen.png)

我这边的版本是在0x3DF8地址处，加上 ASLR 偏移，用 `br s -a xxx` 命令在这个地方下一个断点，然后通过 si 命令进入 dlopen 函数，然后执行到 br x16 的地方，再 si 进入系统库（libdyld.dylib）中的 dlopen 函数。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/dlopen.png)

dlopen 函数会通过 _dyld_func_lookup 找到对应的 dylib 的地址，然后在最后几行命令中的 blr x8 调用该动态库，我们直接在这条指令的下一条指令下断点，然后 c 运行程序跳到该断点。这时候再使用 image list -o -f 命令，就会发现 SubstrateLoader.dylib 动态库已经被加载起来了。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/SubstrateLoader.png)

既然我们已经知道是通过 dlopen 来加载的程序，那么用 IDA 打开 SubstrateLoader.dylib 文件，引用 dlopen 的只有三处：

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/SubstrateLoader_dlopen.png)

可以在这三处都下断点，然后看是哪一处启动了 CydiaSubstrate Mach-O 文件。结果证明是在 InitFunc_0_0 + 0x1AC8 处的 dlopen 调用了 CydiaSubstrate。

在控制台中下断点（SubstrateLoader.dylib 的 ASLR + 0x1AC8），运行程序，然后同样进入 dlopen，在blr x8 的下一条指令处下断点，当程序断在这里时，就会发现我们最终要调试的 CydiaSubstrate 文件终于被加载起来了。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/CydiaSubstrate.png)

然后在 IDA 里面找到 MSHookMessageEx 函数的起点地址（我的版本是0x5908），再下断点，然后运行，就到 MSHookMessageEx 函数里了。

以下是整个流程的动图：

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/动图.gif)

## 逆向分析

### 异同点

通过阅读 IDA 中的汇编代码我们会发现，整个流程以及大部分的代码都与我上一篇博文中讲的老版本的源代码非常相似。

- `sub_14130` 函数是新增的，涉及到调用 “/usr/sbin/aslmanager”、”/usr/lib/system/libsystem_sandbox.dylib” 等动态库，貌似是在做沙盒权限检查（此点存疑）。
- `sub_5E6C`函数即是之前的 MSFindMethod 函数。
- `sub_10000` 函数则做的 if (!direct){…} 这段代码的工作。

很多部分大致相同，这些地方不做过多介绍，接下来主要讲解的是 sub_10000 函数。

### sub_10000

在 IDA 中查看 sub_10000 函数的主要部分（ARM64）：

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/sub_10000.png)

蓝绿色部分是正常执行流程，可以看到主要使用了三个函数： vm_allocate()、vm_deallocate()、vm_remap()。

这里涉及到一个 `trampoline`（蹦床）的概念。在很多时候，我们并不能直接执行我们想要的代码，而需要一个跳转代码或者跳转页面，经过一次或多次的跳转，最后跳到目的代码处执行。生成的这个跳转的代码或者页面就称之为 trampoline。

**在本函数中，target_address 页面就是 trampoline page。**

蹦床通常利用可写代码页（即具有可写、可执行权限）来实现。将指令写入 PROT_EXEC | PROT_WRITE 页面，需要的上下文信息直接包含在生成的代码中。

但是 iOS 中不允许 PROT_EXEC、 PROT_WRITE 这两种权限同时出现在页面中，也就是说不存在可写代码页（在我的测试中，ARMv7是可以的）。那么就需要使用一种替代机制来是实现 trampoline 中特定的上下文数据以及代码。

这种机制就是 **vm_remap() + PC 相对寻址**的组合。

vm_remap() 函数可以在新的地址中映射现有的代码页，并会同时保留页面保护权限（如果映射范围内的内存权限相同，则返回该权限；如果不同，则返回最大限制值）。也就是说，我们可以通过 vm_remap() 函数映射可执行的代码页面，或者可写的数据页面到新的页面地址处。

那么如何配置 trampoline 的数据呢？答案是通过 PC 相对寻址。PC（程序计数器）寄存器指示当前正在执行的指令的地址。我们将可写的数据页映射到可执行的代码页（trampoline page）旁边，然后使用 PC 相对寻址从相邻的可写数据页面加载 trampoline data，这样就达到了“可写”代码页的目的。

在本函数中也是这样做的，**先通过 vm_allocate() 分配两页内存（0x8000），然后通过 vm_deallocate 释放第二页，作为 trampoline page，调用 vm_remap() 将 MSCloseTable 映射到已释放的第二页，最后在第一页（可写数据页）中填充需要的数据，返回第二页（可执行代码页）的首地址。**

以下是我仿照着写的伪代码

```
vm_address_t address = 0x0;
kern_return_t kt;

/* Try to allocate two pages */
kt = vm_allocate (mach_task_self (), &address, PAGE_SIZE*2, 1);
if (kt != KERN_SUCCESS) {
   return ...;
}

/* Now drop the second half of the allocation to make room for the trampoline table */
vm_address_t target_address = address+PAGE_SIZE;
kt = vm_deallocate (mach_task_self (), target_address, PAGE_SIZE);
if (kt != KERN_SUCCESS) {
   return 0;
}

/* Remap the trampoline table to directly follow the address page */
vm_prot_t cur_prot;
vm_prot_t max_prot;

kt = vm_remap (mach_task_self (), &target_address, PAGE_SIZE, 0x0, FALSE, mach_task_self (), (vm_address_t) &MSCloseTable, FALSE, &cur_prot, &max_prot, VM_INHERIT_SHARE);

if (kt != KERN_SUCCESS) {
   return 0;
}

intp64_t **x20 = _class;//&_class+0x8 == &sel
*x20 + 0x8 = sub_5F20
address = x20;
address + 0x8 = (IMP)MSCloseTarget;
	
return target_address;
```

其中 MSCloseTable 的具体内容为：

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/MSCloseTable.png)

MSCloseTarget 中只有三条命令，主要是执行加载、跳转的操作。将PC-0x4000 处的值加载到 x16 寄存器中（这里 IDA 自动将相对偏移-0x4000解释为 _MSCloseTarget，实际上我们只需要这个偏移值，执行时偏移值指向的是 address），然后再将[x16]中的值存入x16，[x16+0x8]中的值存入x17，然后跳到x17所指的地址处执行代码。在动态调试中我们会发现这三条命令再加 nop 指令填充了 trampoline page。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/trampoline.png)

上图中，target_address = 0x105b7c000, address = 0x105b78000，x17 = (IMP)MSCloseTarget，[x16] = &(_class+sel)，[x16]+0x8 = sub_5F20。

也就是说，trampoline 的作用就是使得程序跳转到 MSCloseTarget 处执行，并把参数 _class，sel 的地址以及 sub_5F20 的地址存入 x16 寄存器、[x16]+0x8中。

MSCloseTarget 函数的主要作用是调用 sub_5F20 函数，并传入参数 _class 和 sel。

sub_5F20 函数则是直接调用 class_getMethodImplementation(_class, sel) 函数。

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/sub_5F20.png)

总的来说，sub_10000 函数的作用与老版本的一样，都是为了嵌入一段 class_getMethodImplementation(_class, sel) 的 `JIT (just in time)` 代码，使得程序在运行时才会执行这段代码，而不是在开始的时候就已经传参执行了。

### 为什么不使用原来的 if (!direct)

首先 if (!direct) 中的机器码只定义了 ARM 32位、i386、x86_64位，没有定义 ARM64 的嵌入指令。

~~再者，在 ARM64 中也不能使用这种先 mmap 一段可写的内存，然后将内存改为可执行的权限的方法。~~

![](/images/2017-03-23-逆向知识-Hook-原理之-CydiaSubstrate（二）：MSHookMessageEx/mprotect.png)

上图来自2014年在 The Black Hat USA 会议中的一篇演讲 “[Exploit unpatched iOS vulnerabilities for fun and profit](https://lifeasageek.github.io/papers/jang:ios-slides.pdf)”。

比如以下代码：

```
#import <sys/mman.h>

char shell[] = {0x10, 0x20, 0x70, 0x47}; // return 16;

- (void)viewDidLoad {
    void *page = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_ANON | MAP_PRIVATE, -1, 0);
    if (page == (void*)-1) {
        perror(NULL);
        return;
    }
    memcpy(page, shell, sizeof(shell));
    typedef int (*shell_execute)();
    shell_execute exe = (shell_execute)((int)page+1);
    mprotect(page, 4096, PROT_READ | PROT_EXEC);
    NSString *string = [NSString stringWithFormat:@"%d", exe()];
    NSLog(@"%@", string);
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}
```

在32位的 iOS 系统上，是可以正确运行并打印出16的 ~~但在64位中，则会报错：EXE_BAD_ACCESS (code=257, address=…)~~

补充：测试后发现，在64位 iOS 越狱系统上也是可以使用 mprotect 修改权限的，至于为什么要使用蹦床页面，我猜测是为了更好的兼容性以及后续扩展。

------

## Reference

[1] Implementing imp_implementationWithBlock()　http://landonf.org/2011/04/index.html
[2] pandamonia/libffi-iOS　https://github.com/pandamonia/libffi-iOS/blob/master/patches/ios
[3] WilliamLCobb/iNDS　https://github.com/WilliamLCobb/iNDS/issues/44
[4] 谁偷了我的热更新？Mono，JIT，iOS 　
http://www.cnblogs.com/murongxiaopifu/p/4278947.html
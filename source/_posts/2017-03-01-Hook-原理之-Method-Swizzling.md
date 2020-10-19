---
title: Hook 原理之 Method Swizzling
categories: 基础知识
abbrlink: 3062bfbd
date: 2017-03-01 00:00:00
---

## 基础知识提要

Method Swizzling 其实就是利用了 `runtime` 机制，替换了调用的方法（method）的实现（imp）。很多人一看到有 runtime 就头疼了，跟 runloop 一样，由于被太多大牛提起，反而心生胆怯，觉得是一个很难理解的机制。

runtime 运行时机制，主要是在 OC 和 C 语言（或汇编语言）之间架了一座桥梁。比如我之前的文章 [通过汇编解读 objc_msgSend](https://harpersu00.github.io/77e03b7f.html) 中提到的调用方法的本质是发送消息，方法调用是 OC 中的，而消息发送，找到方法的入口则是 C （或汇编）中的。这其中转换的过程，就是 runtime。

<!-- more -->

runtime 是一个使用 C 语言以及汇编写的动态库。它封装了一些 C 语言的结构体和函数，这些函数可以让使用者在运行时创建、查看、修改类，对象以及方法等。同时也执行着较为底层的传递消息，寻找方法的执行代码的操作。



在使用 runtime 时，一般需要引入 `<objc/runtime.h>` 头文件。

## 实现原理

Method Swizzling，正如之前所提到的，本质就是替换了方法的实现。在 OC 中调用一个方法，这个方法的本质是一条消息，而这条消息的本质，就是一个 selector。也就是说 selector 就代表着这个方法，比如在程序中，我们经常会使用 @selector() 这样的方式来调用另一个方法。

每个类都存储着一个方法列表，又叫调度表（dispatch table），这个表是 selector 与方法的具体实现 IMP 的对应关系，类似于函数名和函数指针。IMP 指向的是方法的具体实现。

![](/images/2017-03-01-基础知识-Hook-原理之-Method-Swizzling/imp1.png)

1. 交换两个方法的实现。也就是说你调用方法 A，但其实是跳到方法 B 的执行代码中。
2. 修改调用方法的类，那么调用 A 类的方法1，其实是调用 B 类的方法1。
3. 直接设置某个方法的 IMP，那么调用这个方法时，也就直接跳到另外一个 IMP 处了。
   ……

总而言之，都是替换了 selector 对应的 IMP。这就是 Method Swizzling 所做的事情。

![](/images/2017-03-01-基础知识-Hook-原理之-Method-Swizzling/imp2.png)

上面两张图均来自[念茜的博客](http://blog.csdn.net/yiyaaixuexi/article/details/9374411)。

概念区分：

> - Selector（typedef struct objc_selector *SEL）:在运行时 Selector 用来代表一个方法的名字。Selector 是一个在运行时被注册（或映射）的C类型字符串。 Selector 由编译器产生并且在当类被加载进内存时由运行时自动进行名字和实现的映射。
> - Method（typedef struct objc_method *Method）:方法是一个不透明的用来代表一个方法的定义的类型。 相当于 SEL + IMP 。
> - Implementation（typedef id (*IMP)(id, SEL,…)）:这个数据类型指向一个方法的实现的最开始的地方。该方法的第一个参数指向调用方法的自身（即内存中类的实例对象，若是调用类方法，该指针则是指向元类对象 metaclass）。第二个参数是这个方法的名字 selector，该方法的真正参数紧随其后。

## 常用方法

1. 通过 SEL 获取一个方法 Method
   `Method class_getInstanceMethod(Class cls, SEL name);`
2. 通过 Method 获取该方法的实现 IMP
   `IMP method_getImplementation(Method m);`
3. 返回一个字符串，描述了方法的参数和返回类型
   `const char * method_getTypeEncoding(Method m);`
4. 通过 SEL 以及 IMP 给一个类添加新的方法 Method，其中 types 就是 method_getTypeEncoding 的返回值。
   `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);`
5. 通过给定的 SEL 替换同一个类中的方法的实现 IMP，其中 SEL 是想要替换的 selector 名，IMP 是替换后的实现。
   `IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);`
6. 交换两个方法的实现 IMP
   `void method_exchangeImplementations(Method m1, Method m2);`

> class_replaceMethod、method_exchangeImplementations 这两个方法的不同之处在于，前者只是将方法 A 的实现替换为方法 B 的实现，而方法 B 的实现并没有改变。后者则是交换了两个方法的实现。

## 示例

Methode Swizzling 有一个常用场景：我想给 app 中每个视图控制器的 viewDidAppear: 方法中添加 log。无论是简单粗暴的给所有视图控制器添加代码，还是通过继承的方式，都会有大量的重复代码出现。

我们可以考虑一种新的方式：在 category 中实现 method swizzling。

```
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        // 取得 SEL
        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        // 取得 Method （对象方法）
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // 如果是类方法的话，使用下面的代码
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

上面的代码中， class_addMethod 方法只是为了作一个判断，检测 self 是否已经有了 originalSelector 方法。如果没有这个方法，就会添加一个 SEL 为 originalSelector 的方法，并将 swizzledSelector 的实现赋给它。接着会进入 if (didAddMethod) 分支。

> 　　这里有一个值得注意的地方，就是如果在 self 中没有实现这个方法，而父类中有实现，那么在 if (didAddMethod) 分支中，其实是**将父类的 originalSelector 的实现赋给 swizzledSelector**，也就是说会调用父类的方法。
> 　　如果父类也没有实现，消息转发也找不到这个方法，那么才是调用之前添加进入 class 的 originalSelector。结果就是 **originalSelector 和 swizzledSelector 的实现均为 xxx_viewWillAppear:** 。

如果 self 中已经有了这个方法，那么 class_addMethod 方法就会失败，直接进入 else 分支交换 IMP。

一个容易让人疑惑的点是：在 xxx_viewWillAppear: 的方法内部又调用了 `[self xxx_viewWillAppear:animated];` 这是因为两个方法的 IMP 已经被调换，这里其实是调用原来的 viewWillAppear: 方法的实现。

在这个例子中，虽然可以不用 Method Swizzling 方法，直接在 category 中重写 viewWillAppear: 方法也能达到目的。但是前者可以控制执行的顺序，以及可以用在非系统类中。而 category 中的方法是直接覆盖了原来的方法的，调用顺序是既定的，且只能用在系统类中。

## 注意事项

### +load

一般来说，Method Swizzling 应该只在 `+load` 方法中完成。 在 Objective-C 的运行时中，每个类都会自动调用两个方法。+load 是在一个类被初始装载时调用的，+initialize 是在应用第一次调用该类的类方法或实例方法前调用的。在应用程序的一开始就调用执行，是最安全的，避免了很多并发、异常等问题。如果在 +initialize 初始化方法中调用，runtime 很可能死于一个诡异的状态。

### dispatch_once

由于 swizzling 改变了全局的状态，所以需要确保在运行时，我们采用的预防措施是可用的。原子操作就是这样一个用于确保代码只会被执行一次的预防措施，就算是在不同的线程中也能确保代码只执行一次。Grand Central Dispatch 的 dispatch_once 满足了这些需求，所以，**Method Swizzling 应该在 dispatch_once 中完成**。

### 调用原始实现

由于很多内部实现对我们来说是不可见的，使用方法交换可能会导致代码结构的改变，而对程序产生其他影响，因此应该调用原始实现来保证内部操作的正常运行。

### 注意命名

这也是方法命名的规则，给需要转换的方法加前缀，以区别于原生方法。

### 类簇

Method Swizzling对NSArray、NSMutableArray、NSDictionary、NSMutableDictionary 等这些类簇是不起作用的。因为这些类簇类，其实是一种抽象工厂的设计模式。抽象工厂内部有很多其它继承自当前类簇的子类，抽象工厂类会根据不同情况，创建不同的抽象对象来进行使用，真正执行操作的并不是类簇本身。

那么要使它有作用，就需要使用类簇里面的真正的类，比如 `objc_getClass("__NSArrayI")`

|        类簇         |    常用子类     |
| :-----------------: | :-------------: |
|       NSArray       |   __NSArrayI    |
|   NSMutableArray    |   __NSArrayM    |
|    NSDictionary     | __NSDictionaryI |
| NSMutableDictionary | __NSDictionaryM |

------

## Reference

[1] iOS黑魔法－Method Swizzling　http://www.jianshu.com/p/ff19c04b34d0
[2] iOS runtime实战应用：Method Swizzling　http://www.jianshu.com/p/3efc3e94b14c
[3] 黑魔法 - Method Swizzling　http://nucleardev.com/Method-Swizzling/
[4] Method Swizzling　http://nshipster.cn/method-swizzling/
---
title: 通过汇编解读 objc_msgSend
categories: 逆向知识
abbrlink: 77e03b7f
date: 2016-11-09
---

## 基础知识提要

调用方法，本质是发送消息。比如：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div></pre></td><td class="code"><pre><div class="line">Person *p = [[Person alloc] init];</div><div class="line">[p test];</div><div class="line"></div><div class="line"><span class="comment">// 本质是发送消息： clang -rewrite-objc main.m</span></div><div class="line">((<span class="keyword">void</span> (*)(<span class="keyword">id</span>, SEL))(<span class="keyword">void</span> *)objc_msgSend)((<span class="keyword">id</span>)p, sel_registerName(<span class="string">"test"</span>));</div></pre></td></tr></tbody></table>

<!-- more -->

当编译器遇到一个方法调用时，它会将方法的调用翻译成以下函数中的一个，  
objc\_msgSend、 objc\_msgSend\_stret、 objc\_msgSendSuper 和 objc\_msgSendSuper\_stret。

> - 发送给对象的父类的消息会使用 objc\_msgSendSuper ;
> - 有数据结构作为返回值的方法会使用 objc\_msgSendSuper\_stret 或 objc\_msgSend\_stret ;
> - 其它的消息都是使用 objc\_msgSend 发送的。

也就是说所有的方法调用，都是通过 objc\_msgSend（或其大类）来实现转发的。

objc\_msgSend 的具体实现由汇编语言编写而成，不同平台有不同的实现，objc-msg-arm.s、objc-msg-arm64.s、objc-msg-i386.s、objc-msg-simulator-i386.s、objc-msg-simulator-x86\_64.s、objc-msg-x86\_64.s。  
本文以 ARM64 平台为例。

## 汇编分析

### 汇编概览

如下图所示：

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend1.png)

### 流程图分析

#### 分支1：X0 = 0

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend2.png)

这条分支很简单，对照图1的总图来讲，就是蓝色的那条线，第1行->第2行-> 29 \-> 35\~41 ret。  
先对传入的 X0（即对象地址）作判断，如果 X0=0，则直接返回。

#### 分支2：X0 \< 0 \(Tagger Pointer\)

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend3.png)

对照图1来讲，流程为黄色的那根线，1\~2 \-> 29\~34 \-> 6 \-> …

判断 X0\<0，即地址最高位为1，这是 Tagger Pointer 类型的标志（对于 ARM64 架构来讲），关于这个类型，部分内容在我之前的文章[copy 与 mutableCopy（传说中的深浅拷贝）](https://harpersu00.github.io/accb5a79.html)中5.4节有提到。

loc\_1800b9c30 这个模块取出了 Tagger Pointer 的类索引表，赋值给 X10。  
下一行 `UBFM X11,X0,#0x3C,#0x3F`，取 0x3C\~0x3F 中的值赋给 X11，其余位以0填充，与图1第32行的意思相同，都是取出最高4位，比如 NSString 类型的 Tagger Pointer 最高4位为 a，运算过后，x11 = 0xa 。  
接着 `LDR X9,[X10,X11,LSL#3]`，先运算 X11 左移3位等于 0x50。x9 = x10\[0x50\]，也就是在类索引表中查找所属类。找到后跳到 loc\_1800b9BD0，也就是图1中的第6行。

#### 分支3：X0 > 0

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend4.png)

这是大多数情况会走的流程。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div></pre></td><td class="code"><pre><div class="line"><span class="comment">//类的结构</span></div><div class="line"><span class="keyword">struct</span> objc_class : objc_object {</div><div class="line">    <span class="comment">// Class ISA;       //继承自objc_object</span></div><div class="line">    Class superclass; 	<span class="comment">// 父类引用</span></div><div class="line">    cache_t cache;		<span class="comment">// 用来缓存指针和虚函数表</span></div><div class="line">    class_data_bits_t bits; <span class="comment">// class_rw_t 指针加上 rr/alloc 标志</span></div><div class="line">}</div></pre></td></tr></tbody></table>

接下来我们根据汇编指令一条条来分析。  
`LDR X13,[X0]` 取出调用方法的对象指针保存的地址（从上面代码可以看出，就是 isa 指针地址），赋给 X13。

`AND X9,X13,#0x1FFFFFFF8` 解读这条指令之前，要先了解 isa 指针的结构。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div></pre></td><td class="code"><pre><div class="line"><span class="keyword">union</span> isa_t {</div><div class="line">    isa_t() { }</div><div class="line">    isa_t(uintptr_t value) : bits(value) { }</div><div class="line">    Class cls;</div><div class="line">    uintptr_t bits;</div><div class="line">    <span class="keyword">struct</span> {</div><div class="line">        uintptr_t indexed           : <span class="number">1</span>;</div><div class="line">        uintptr_t has_assoc         : <span class="number">1</span>;</div><div class="line">        uintptr_t has_cxx_dtor      : <span class="number">1</span>;</div><div class="line">        uintptr_t shiftcls          : <span class="number">33</span>; </div><div class="line">        uintptr_t magic             : <span class="number">6</span>;</div><div class="line">        uintptr_t weakly_referenced : <span class="number">1</span>;</div><div class="line">        uintptr_t deallocating      : <span class="number">1</span>;</div><div class="line">        uintptr_t has_sidetable_rc  : <span class="number">1</span>;</div><div class="line">        uintptr_t extra_rc          : <span class="number">19</span>;</div><div class="line">    };</div><div class="line">};</div></pre></td></tr></tbody></table>

首先先来看一下这 64 个二进制位每一位的含义：

| 区域名                      |                           代表信息                           |
| --------------------------- | :----------------------------------------------------------: |
| indexed \(0位\)             |     0 表示普通的 isa 指针，1 表示使用优化，存储引用计数      |
| has\_assoc \(1位\)          |       表示该对象是否有关联引用，如果没有，则析构时更快       |
| has\_cxx\_dtor \(2位\)      | 表示该对象是否有 C++ 或 ARC 的析构函数，如果没有，则析构时更快 |
| shiftcls \(3\~35位\)        |                           类的指针                           |
| magic \(36\~41位\)          |         固定值，用于在调试时分辨对象是否未完成初始化         |
| weakly\_referenced \(42位\) |     表示该对象是否有过 weak 对象，如果没有，则析构时更快     |
| deallocating \(43位\)       |                    表示该对象是否正在析构                    |
| has\_sidetable\_rc \(44位\) |      表示该对象的引用计数值是否过大无法存储在 isa 指针       |
| extra\_rc \(45\~63位\)      |                  存储引用计数值减一后的结果                  |

也就是说 0x1FFFFFFF8 取1的位数刚好是 shiftcls 的区域，是 isa 指针中存储的该对象的类指针。所以 X9 = isa->cls。

`LDP X10,X11,[X9,#0X10]`： X9+16个字节，也就是跳过了8个字节的 isa 指针，和8个字节的 superclass 指针，到了 cache 指针这里。 cache 的结构如下：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div></pre></td><td class="code"><pre><div class="line"><span class="keyword">struct</span> bucket_t {</div><div class="line">    <span class="keyword">void</span> *sel;</div><div class="line">    <span class="keyword">void</span> *imp;</div><div class="line">};</div><div class="line"></div><div class="line"><span class="keyword">struct</span> cache_t {</div><div class="line">    <span class="keyword">struct</span> bucket_t *buckets;</div><div class="line">    mask_t mask;</div><div class="line">    mask_t occupied;</div><div class="line">};</div></pre></td></tr></tbody></table>

因此，X10=buckets 指针，X11 的低32位为 mask，高32位为 occupied（mask\_t 是 int 类型）。 occupied是 cache 中实际拥有的方法个数。

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend5.png)

`AND W12,W1,W11`： 将 \_cmd 的低32位和 cache->mask 作与运算。  
`ADD X12,X10,X12,LSL#4`: 与运算后的结果左移4位，作为buckets的索引（相当于数组下标）。这里也可以看出 mask 的作用，应该是一种优化的 hash 表搜索算法。将取得的指针赋给 X12。  
`LDP X16,X17,[X12]`： 由 bucket 的结构可以知道，这里是将 bucket \[\(\_cmd\&mask\)\<\<4\] 中的 sel 赋给 X16，imp 赋给 X17（imp 为方法的入口地址）。  
这三条指令就是通过 mask 找到一个 bucket 元素。

`CMP X16,X1`, `B.NE loc_1800B9BEC`, `BR X17`： 这3条指令很好理解，比较 bucket 元素中的 sel 和 \_cmd 的值是否相等，不相等，则跳到 loc\_1800B9BEC 模块，相等则直接进入对应 imp（方法入口地址）。

`loc_1800B9BEC CBZ X16,_objc_msgSend_uncached_impcache`： 如果 X16=0 则跳到 objc\_msgSend\_uncached 这个函数去，不等于0则继续执行。  
`CMP X12,X10`, `B.EQ loc_1800B9C00`： 判断是否已搜索到最后一个 bucket（即 bucket 的初始地址），是则跳到 loc\_1800B9C00，否则继续执行。

- 先讨论没有搜索完的情况，  
  `loc_1800B9C00 LDP X16,X17,[X12,#-0X10]`, `B loc_1800B9BE0`： bucket 元素减16字节，即跳到前一个 bucket 元素，同样将 sel 和 imp 指针赋值，然后跳回与 \_cmd 比较的那条指令循环。

- 直到搜索完毕，  
  `ADD X12,X12,W11,UXTW #4`： x12 = buckets+\(mask\<\<4\)，扩大搜索范围，在缓存内全面搜索。（进行到这一步，说明 bucket \[\(\_cmd\&mask\)\<\<4\] 元素之前的 bucket 已全部被占满，且均不是我们要找的方法）  
  `LDP X16,X17,[X12]`： 跟之前的命令意思一样。

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend6.png)

可以看到，之后的流程跟前面的循环一模一样，但是加大了搜索范围，从 bucket \[mask\<\<4\] 往前开始搜索（进行到这一步说明 bucket \[\(\_cmd\&mask\)\<\<4\] 前面的缓存都占满了）。从以上分析，我们可以看出，**能在缓存 cache 里找到的方法，会直接跳到入口地址 X17。**而没有在 cache 里的方法，则要继续调用 objc\_msgSend\_uncached 函数。现在，返回图1再查看，是不是觉得思路清晰很多呀！

#### 关于缓存 cahce

cache 的原则是缓存那些可能要执行的函数地址。

> 有一种说法是，只要函数执行过一次的方法，都会存入缓存。但在我的测试中，有时候会遵循这种说法，有时候又不尽然，执行过的方法不一定会被放入缓存，但没有被执行过的肯定不会进入缓存。具体什么样的操作会导致方法被载入缓存，还需要从类的初始化探讨起，此点存疑。

cahce 其实是一个 hash 表，通过 \_cmd\&mask 的结果再左移4位，作为索引值，如果这个地址存的方法 \_cmd2 与 \_cmd 不同，那么有两种原因：一是 \_cmd 压根儿没被载入缓存；二是由于它的索引值跟 \_cmd 相同，但 \_cmd2 先进入缓存，因此 \_cmd2 占据了这个位置。这时，如果 \_cmd 被载入缓存的话，则在 \_cmd2 索引值-1的位置存入，如果这个位置也不为0，那么继续前往索引值-2的位置，直到找到一个0位，然后存入。

在上面的汇编分析中，我们也能看到这个思路。在图1中第8行，取 bucket 索引值；第10行，比较 \_cmd 值；如果不同则第13行，查看是否为0，如果为0，则不再搜索，直接进入 uncache 函数（因为是0的话，由上一段分析可以知道，说明这个方法没有在缓存里）；如果不为0，则前往索引值-1（地址-16）的位置查找；第17行返回循环到第10行。

下面来做一个测试，

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div></pre></td><td class="code"><pre><div class="line"><span class="class"><span class="keyword">@implementation</span> <span class="title">aboutObjectiveC</span></span></div><div class="line">-(<span class="keyword">void</span>)objc_msgSend1 {</div><div class="line">	<span class="built_in">NSLog</span>(<span class="string">@"objc_msgSend1"</span>);</div><div class="line">	[<span class="keyword">self</span> objc_msgSend2];</div><div class="line">}</div><div class="line"></div><div class="line">-(<span class="keyword">void</span>)objc_msgSend1 {</div><div class="line">	<span class="built_in">NSLog</span>(<span class="string">@"objc_msgSend2"</span>);</div><div class="line">}</div></pre></td></tr></tbody></table>

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend7.png)

如上图所示，在 main.m 第17行下断点（即第二次执行 objc\_msgSend1 方法时），si 进入 objc\_msgSend 函数，然后执行到图1中的第7行，打印各值如下

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/objc_msgSend8.png)

w11 是 mask 的值为0011，跟 init的 SEL\(0x1883910b6\) 指针作与运算，为0x2，左移4位为0x20，因此在 x10+0x20 处载入 cache；跟 objc\_msgSend1 的 SEL\(0x10008ecac\) 作与运算，为0x0，左移4位还是0x0，因此在 x10 bucket 处载入 cache；同样对 objc\_msgSend2 作与运算左移4位，也是0x20，而 bucket\[0x20\] 处已经被 init 占用了，因此前往 bucket\[0x20-0x10\] 处，这个位置是0，所以将 objc\_msgSend2 填入缓存的这个位置。如下图所示：

![](/images/2016-11-09-逆向知识-通过汇编解读-objc_msgSend/cache.png)

#### lookUpImpOrForward 函数

我们已经知道如果缓存中没有找到该方法，则跳转执行 \_objc\_msgSend\_uncached\_impcache，在这里又会执行 bl \_class\_lookupMethodAndLoadCache3 指令，跳转到 \_class\_lookupMethodAndLoadCache3，由汇编语言的实现回到了 C 函数的实现，这个函数只是简单的调用了另外一个函数 lookUpImpOrForward，并传入参数 cache=NO，这个函数是 Runtime 消息机制中非常重要的一环。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div><div class="line">23</div><div class="line">24</div><div class="line">25</div><div class="line">26</div><div class="line">27</div><div class="line">28</div><div class="line">29</div><div class="line">30</div><div class="line">31</div><div class="line">32</div><div class="line">33</div><div class="line">34</div><div class="line">35</div><div class="line">36</div><div class="line">37</div><div class="line">38</div><div class="line">39</div><div class="line">40</div><div class="line">41</div><div class="line">42</div><div class="line">43</div><div class="line">44</div><div class="line">45</div><div class="line">46</div><div class="line">47</div><div class="line">48</div><div class="line">49</div><div class="line">50</div><div class="line">51</div><div class="line">52</div><div class="line">53</div><div class="line">54</div><div class="line">55</div><div class="line">56</div><div class="line">57</div><div class="line">58</div><div class="line">59</div><div class="line">60</div><div class="line">61</div><div class="line">62</div><div class="line">63</div><div class="line">64</div><div class="line">65</div><div class="line">66</div><div class="line">67</div><div class="line">68</div><div class="line">69</div><div class="line">70</div><div class="line">71</div><div class="line">72</div><div class="line">73</div><div class="line">74</div><div class="line">75</div><div class="line">76</div><div class="line">77</div><div class="line">78</div><div class="line">79</div><div class="line">80</div><div class="line">81</div><div class="line">82</div><div class="line">83</div><div class="line">84</div><div class="line">85</div><div class="line">86</div><div class="line">87</div><div class="line">88</div><div class="line">89</div><div class="line">90</div><div class="line">91</div><div class="line">92</div><div class="line">93</div><div class="line">94</div><div class="line">95</div><div class="line">96</div><div class="line">97</div><div class="line">98</div><div class="line">99</div><div class="line">100</div><div class="line">101</div><div class="line">102</div><div class="line">103</div><div class="line">104</div></pre></td><td class="code"><pre><div class="line">IMP lookUpImpOrForward(Class cls, SEL sel, <span class="keyword">id</span> inst, </div><div class="line">                       <span class="keyword">bool</span> initialize, <span class="keyword">bool</span> cache, <span class="keyword">bool</span> resolver)</div><div class="line">{</div><div class="line">    Class curClass;</div><div class="line">    IMP imp = <span class="literal">nil</span>;</div><div class="line">    Method meth;</div><div class="line">    <span class="keyword">bool</span> triedResolver = <span class="literal">NO</span>;</div><div class="line"></div><div class="line">    runtimeLock.assertUnlocked();</div><div class="line"></div><div class="line">	<span class="comment">//因为 _class_lookupMethodAndLoadCache3 传入的 cache = NO，</span></div><div class="line">	<span class="comment">//所以这里会直接跳过 if 中代码的执行，</span></div><div class="line">	<span class="comment">//在 objc_msgSend 中已经使用汇编代码查找过了。 </span></div><div class="line"></div><div class="line">    <span class="keyword">if</span> (cache) {</div><div class="line">        imp = cache_getImp(cls, sel);</div><div class="line">        <span class="keyword">if</span> (imp) <span class="keyword">return</span> imp;</div><div class="line">    }</div><div class="line"></div><div class="line">	<span class="comment">//根据 cls-&gt;isRealized() 来判断是否要调用 realizeClass 函数在</span></div><div class="line">	<span class="comment">// Objective-C 运行时 初始化的过程中会对其中的类进行第一次初始化</span></div><div class="line">	<span class="comment">//也就是执行 realizeClass 方法，为类分配可读写结构体 class_rw_t</span></div><div class="line">	<span class="comment">//的空间，并返回正确的类结构体。</span></div><div class="line">    <span class="keyword">if</span> (!cls-&gt;isRealized()) {</div><div class="line">        rwlock_writer_t lock(runtimeLock);</div><div class="line">        realizeClass(cls);</div><div class="line">    }</div><div class="line"></div><div class="line">	<span class="comment">//根据 cls-&gt;isInitialized() 来判断类的是不是 initialized，</span></div><div class="line">	<span class="comment">//也就是类的首次被使用的时候，其 initialize 方法要在此时被调用</span></div><div class="line">	<span class="comment">//一次，也仅此一次。没有 initialized 的话，则调用</span></div><div class="line">	<span class="comment">//_class_initialize 函数去触发这个类的 initialize 方法，然后</span></div><div class="line">	<span class="comment">//会设置 isInitialized 状态为 initialized </span></div><div class="line">    <span class="keyword">if</span> (initialize  &amp;&amp;  !cls-&gt;isInitialized()) {</div><div class="line">        _class_initialize (_class_getNonMetaClass(cls, inst));</div><div class="line">        <span class="comment">// If sel == initialize, _class_initialize will send +initialize and </span></div><div class="line">        <span class="comment">// then the messenger will send +initialize again after this </span></div><div class="line">        <span class="comment">// procedure finishes. Of course, if this is not being called </span></div><div class="line">        <span class="comment">// from the messenger then it won't happen. 2778172</span></div><div class="line">    }</div><div class="line"></div><div class="line">    <span class="comment">// The lock is held to make method-lookup + cache-fill atomic </span></div><div class="line">    <span class="comment">// with respect to method addition. Otherwise, a category could </span></div><div class="line">    <span class="comment">// be added but ignored indefinitely because the cache was re-filled </span></div><div class="line">    <span class="comment">// with the old value after the cache flush on behalf of the category.</span></div><div class="line"> retry:</div><div class="line">    runtimeLock.read();</div><div class="line"></div><div class="line">    <span class="comment">// 是否开启GC(垃圾回收)； 新版本这一段代码已经没有了。</span></div><div class="line">    <span class="keyword">if</span> (ignoreSelector(sel)) {</div><div class="line">        imp = _objc_ignored_method;</div><div class="line">        cache_fill(cls, sel, imp, inst);</div><div class="line">        <span class="keyword">goto</span> done;</div><div class="line">    }</div><div class="line"></div><div class="line">    <span class="comment">// 这里再次查找 cache 是因为有可能 cache 真的又有了，因为锁的原因</span></div><div class="line">    imp = cache_getImp(cls, sel);</div><div class="line">    <span class="keyword">if</span> (imp) <span class="keyword">goto</span> done;</div><div class="line"></div><div class="line">    <span class="comment">// Try this class's method lists.</span></div><div class="line">    meth = getMethodNoSuper_nolock(cls, sel);</div><div class="line">    <span class="keyword">if</span> (meth) {</div><div class="line">        log_and_fill_cache(cls, meth-&gt;imp, sel, inst, cls);</div><div class="line">        imp = meth-&gt;imp;</div><div class="line">        <span class="keyword">goto</span> done;</div><div class="line">    }</div><div class="line"></div><div class="line">    <span class="comment">// Try superclass caches and method lists.</span></div><div class="line">    curClass = cls;</div><div class="line">    <span class="keyword">while</span> ((curClass = curClass-&gt;superclass)) {</div><div class="line">        <span class="comment">// Superclass cache.</span></div><div class="line">        imp = cache_getImp(curClass, sel);</div><div class="line">        <span class="keyword">if</span> (imp) {</div><div class="line">            <span class="keyword">if</span> (imp != (IMP)_objc_msgForward_impcache) {</div><div class="line">                <span class="comment">// Found the method in a superclass. Cache it in this class.</span></div><div class="line">                log_and_fill_cache(cls, imp, sel, inst, curClass);</div><div class="line">                <span class="keyword">goto</span> done;</div><div class="line">            }</div><div class="line">            <span class="keyword">else</span> {</div><div class="line">                <span class="comment">// Found a forward:: entry in a superclass.</span></div><div class="line">                <span class="comment">// Stop searching, but don't cache yet; call method </span></div><div class="line">                <span class="comment">// resolver for this class first.</span></div><div class="line">                <span class="keyword">break</span>;</div><div class="line">            }</div><div class="line">        }</div><div class="line"></div><div class="line">        <span class="comment">// Superclass method list.</span></div><div class="line">        meth = getMethodNoSuper_nolock(curClass, sel);</div><div class="line">        <span class="keyword">if</span> (meth) {</div><div class="line">            log_and_fill_cache(cls, meth-&gt;imp, sel, inst, curClass);</div><div class="line">            imp = meth-&gt;imp;</div><div class="line">            <span class="keyword">goto</span> done;</div><div class="line">        }</div><div class="line">    }</div><div class="line"></div><div class="line">    <span class="comment">// No implementation found. Try method resolver once.</span></div><div class="line">    <span class="keyword">if</span> (resolver  &amp;&amp;  !triedResolver) {</div><div class="line">        runtimeLock.unlockRead();</div><div class="line">        _class_resolveMethod(cls, sel, inst);</div><div class="line">        <span class="comment">// Don't cache the result; we don't hold the lock so it may have </span></div><div class="line">        <span class="comment">// changed already. Re-do the search from scratch instead.</span></div><div class="line">        triedResolver = <span class="literal">YES</span>;</div><div class="line">        <span class="keyword">goto</span> retry;</div><div class="line">    }</div><div class="line"></div><div class="line">    <span class="comment">// No implementation found, and method resolver didn't help. </span></div><div class="line">    <span class="comment">// Use forwarding.</span></div><div class="line">    imp = (IMP)_objc_msgForward_impcache;</div><div class="line">    cache_fill(cls, sel, imp, inst);</div><div class="line"></div><div class="line"> done:</div><div class="line">    runtimeLock.unlockRead();</div><div class="line"></div><div class="line">    <span class="comment">// paranoia: look for ignored selectors with non-ignored implementations （新版本没有这两句断言）</span></div><div class="line">    assert(!(ignoreSelector(sel)  &amp;&amp;  imp != (IMP)&amp;_objc_ignored_method));</div><div class="line"></div><div class="line">    <span class="comment">// paranoia: never let uncached leak out （新版本没有这两句断言）</span></div><div class="line">    assert(imp != _objc_msgSend_uncached_impcache);</div><div class="line"></div><div class="line">    <span class="keyword">return</span> imp;</div><div class="line">}</div></pre></td></tr></tbody></table>

lookUpImpOrForward 主要做了以下几个工作

- 判断类的初始化 cls->isRealized\(\) 和 cls->isInitialized\(\) ；
- 是否开启GC\(垃圾回收\)；（新版本没有这一步）
- 再次尝试去缓存中获取IMP；（因为锁的原因）
- 找不到接着去 class 的方法列表查找，找到会加入缓存列表然后返回 IMP；
- 找不到，去父类的缓存列表找，然后去父类的方法列表找，找到了会加入自己的缓存列表，然后返回 IMP，找不到循环此步骤，直到找到基类；
- 都找不到则 \_class\_resolveMethod 函数会被调用，进入消息动态处理、转发阶段。

对于 objc\_msgSend 反汇编的分析就结束啦！如果是在动态调试过程中，遇到 objc\_msgSend 想要进入被调用的方法的话，有 cache，则直接 si 进入 br X17，如果没有 cache，则在 \_objc\_msgSend\_uncached\_impcache 函数中最后几行中的 br X17 指令输入 si 即可进入被调用方法。

---

## Reference

\[1\] ObjC Runtime（五）：消息传递机制　<https://xiuchundao.me/post/runtime-messaging>  
\[2\] 从源代码看 ObjC 中消息的发送　<http://draveness.me/message/>  
\[3\] objc\_msgSend内部到底做了什么？　<http://oriochan.com/14710029019312.html>  
\[4\] 用 isa 承载对象的类信息　<http://www.desgard.com/isa/>  
\[5\] 深入解析 ObjC 中方法的结构  
[https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/\%E6\%B7\%B1\%E5\%85\%A5\%E8\%A7\%A3\%E6\%9E\%90\%20ObjC\%20\%E4\%B8\%AD\%E6\%96\%B9\%E6\%B3\%95\%E7\%9A\%84\%E7\%BB\%93\%E6\%9E\%84.md](<https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/深入解析 ObjC 中方法的结构.md>)
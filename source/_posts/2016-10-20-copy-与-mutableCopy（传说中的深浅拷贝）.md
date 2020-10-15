---
title: copy 与 mutableCopy（传说中的深浅拷贝）
date: 2016-10-20
categories: 编程语言语法
---

## 概念

对象拷贝有两种方式：浅拷贝和深拷贝。

浅拷贝\(shallow copy\)，并不拷贝对象本身，**仅仅是拷贝指向对象的指针**。  
如果 `B = [A 浅拷贝]`，则 A、B 两个对象中都保存的同一个指针，如果 A 通过这个指针改变了指针指向的对象，那么 B 指针指向的对象也就随之改变了；

深拷贝是直接**拷贝整个对象到另一块内存中**，开辟新的地址来存储，两个对象至此一别，再无关联。

<!-- more -->

> 对于集合对象（如 NSSArray、NSDictionary 等）而言，又有**单层深拷贝**与**完全拷贝**之分。
>
> 单层深拷贝\(one-level-deep copy\)：指的是对于被拷贝对象，至少有一层是深拷贝。  
> 完全拷贝\(real-deep copy\)：指的是对于被拷贝对象的每一层都是对象拷贝。

## copy 与 mutableCopy

不管是集合类对象，还是非集合类对象，接收到 copy 和 mutableCopy 消息时，都遵循以下准则：

- copy 返回不可变\(imutable\)对象，如果对copy返回值使用可变对象方法就会crash；
- mutablCopy 默认返回可变\(mutable\)对象（如果拷贝后的对象本身是不可变的，那也没法变呀，总不能改变人对象的类型吧，比如 `NSString *str2 = [str1 mutableCopy];` ）。

## 示例头文件

首先定义了一系列会用到的属性，另外，我在宏定义里去掉了 NSLog 的时间戳，然后定义了 AmyLog ，用来显示对象的所属类，以及地址。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div></pre></td><td class="code"><pre><div class="line"><span class="meta">#import <span class="meta-string">&lt;Foundation/Foundation.h&gt;</span></span></div><div class="line"></div><div class="line"><span class="meta">#define NSLog(FORMAT, ...) fprintf(stderr, <span class="meta-string">"%s\n"</span>, [[NSString stringWithFormat:FORMAT, ##__VA_ARGS__] UTF8String] )</span></div><div class="line"></div><div class="line"><span class="meta">#define AmyLog(_var) NSLog(@<span class="meta-string">"     (%@ *) %p\n"</span>, [_var class], _var)</span></div><div class="line"></div><div class="line"><span class="class"><span class="keyword">@interface</span> <span class="title">FirstClass</span> : <span class="title">NSObject</span></span></div><div class="line"></div><div class="line"><span class="comment">//非集合类对象</span></div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSString</span> *string;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">strong</span>) <span class="built_in">NSMutableString</span> *mString;</div><div class="line"></div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSString</span> *stringCopy;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSString</span> *stringMutableCopy;</div><div class="line"></div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">strong</span>) <span class="built_in">NSMutableString</span> *mStringCopy;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">strong</span>) <span class="built_in">NSMutableString</span> *mStringMutableCopy;</div><div class="line"></div><div class="line"><span class="comment">//集合类对象</span></div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSArray</span> *array;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSArray</span> *arrayCopy;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">copy</span>) <span class="built_in">NSArray</span> *arrayMutableCopy;</div><div class="line"></div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">strong</span>) <span class="built_in">NSMutableArray</span> *mArrayCopy;</div><div class="line"><span class="keyword">@property</span> (<span class="keyword">nonatomic</span>, <span class="keyword">strong</span>) <span class="built_in">NSMutableArray</span> *mArrayMutableCopy;</div><div class="line"></div><div class="line"><span class="keyword">@end</span></div></pre></td></tr></tbody></table>

## 非集合类对象\(NSString\)

### 执行代码：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div></pre></td><td class="code"><pre><div class="line">FirstClass *fC = [[FirstClass alloc] init];</div><div class="line">        </div><div class="line">fC.string = <span class="string">@"originString"</span>;</div><div class="line">fC.stringCopy = fC.string;  <span class="comment">//浅，指针 (不可变String）</span></div><div class="line">fC.stringMutableCopy = [fC.string mutableCopy];  <span class="comment">//深，新地址 (可变String)</span></div><div class="line">        </div><div class="line">fC.mStringCopy = [fC.string <span class="keyword">copy</span>];  <span class="comment">//浅，指针 (不可变String）</span></div><div class="line">fC.mStringMutableCopy = [fC.string mutableCopy];  <span class="comment">//深，新地址 (可变String)</span></div><div class="line">        </div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"\n非集合类对象(NSString)：\noriginal address:    "</span>); AmyLog(fC.string);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSString:    "</span>); AmyLog(fC.stringCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSString:    "</span>); AmyLog(fC.stringMutableCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSMutableString:   "</span>); AmyLog(fC.mStringCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSMutableString:    "</span>); AmyLog(fC.mStringMutableCopy);</div></pre></td></tr></tbody></table>

### 打印结果：

![](/images/2016-10-20-语法-copy-与-mutableCopy（传说中的深浅拷贝）/NSString.png)

### \_\_NSCFConstantString 和 \_\_NSCFString

**\_\_NSCFConstantString**

> \_\_NSCFConstantString 对象，就是**字符串常量对象，存储在栈上**，创建之后由系统来管理内存释放.相同内容的 NSCFConstantString 对象地址相同。该对象引用计数很大，为固定值不会变化，表示无限运行的 retainCount ，对其进行 retain 或 release 也不会影响其引用计数。
>
> 当创建一个 NSCFConstantString 对象时，会检测这个字符串内容是否已经存在，如果存在，则直接将地址赋值给变量；不存在的话，则创建新地址，再赋值。
>
> 总的来说，**对于 NSCFConstantString 对象，只要字符串内容不变，就不会分配新的内存地址**，无论你是赋值、 retain、 copy 。这种优化在大量使用 NSString 的情况下可以节省内存，提高性能。
>
> ——摘自简书作者路子：[NSString：内存简述，Copy与Strong关键字](http://www.jianshu.com/p/0e98f37114e3)

对于 NSString 来说，以下几种赋值方法将会保存为 NSCFConstantString 对象：

1.  直接赋值，如 `NSString *str = @"STR";`
2.  stringWithString ,如 `NSString *str = [NSString stringWithString:@"Str"];`
3.  `str1 = str2;`
4.  `str1 = [str2 copy/retain];`

**\_\_NSCFString**

\_\_NSCFString 对象是 **NSString 的一种子类，存储在堆上**，不属于字符串常量对象。该对象创建之后和其他的 Obj 对象一样引用计数为1，对其执行 retain 和 release 将改变其 retainCount 。

诸如 `[NSString stringWithFormat:]` 方法以及 `NSMutableString` 创建的字符串等，都是构造的这种对象。

### 分析

`mutableCopy` 意味着你告诉编译器，我拷贝过来的这个对象可能会改变，因此编译器肯定会新开辟一个地址给你。 因此采用这种方式的都是深拷贝（包括单层深拷贝和完全拷贝）。  
通过结果我们也可以看见，正如我们前面所提到的，copy 返回不可变对象，因此对于原始对象是不可变的 NSSring 类型，完全没有必要再新分配一块内存。**因此对于不可变的非集合对象，采用 mutableCopy 方式的拷贝就是深拷贝，copy 是浅拷贝。**

## 非集合类对象\(NSMutableString\)

### 执行代码：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div></pre></td><td class="code"><pre><div class="line">fC.mString = [<span class="built_in">NSMutableString</span> stringWithString:<span class="string">@"mStringingi"</span>];</div><div class="line">fC.stringCopy = fC.mString;  <span class="comment">//深，新地址（可变String）</span></div><div class="line">fC.stringMutableCopy = [fC.mString mutableCopy]; <span class="comment">////深，新地址 (可变String)</span></div><div class="line">        </div><div class="line">fC.mStringCopy = [fC.mString <span class="keyword">copy</span>];  <span class="comment">//深，新地址，可变String）</span></div><div class="line">fC.mStringMutableCopy = [fC.mString mutableCopy]; <span class="comment">//深，新地址，(可变String)</span></div><div class="line">        </div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"\n非集合类对象(NSMutableString)：\noriginal address:    "</span>); AmyLog(fC.mString);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSString:    "</span>); AmyLog(fC.stringCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSString:    "</span>); AmyLog(fC.stringMutableCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSMutableString:   "</span>); AmyLog(fC.mStringCopy);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSMutableString:    "</span>); AmyLog(fC.mStringMutableCopy);</div></pre></td></tr></tbody></table>

### 打印结果：

![](/images/2016-10-20-语法-copy-与-mutableCopy（传说中的深浅拷贝）/NSMutableString.png)

### 分析

我们已经知道只要用 `mutableCopy` ，对于非集合对象的拷贝，无论可变不可变，都是深拷贝。 `copy` 对于不可变对象的拷贝是浅拷贝。那么对于 `copy` 可变对象呢？如上图的结果所示，是深拷贝。

也很容易理解，我这个对象是可变的，我随时可能通过其他引用它的指针来改变这个对象，现在你要拷贝一份不可变的内容，编译器当然不能直接把它的指针给你啦，这样岂不就是可变的了？所以要新分配给你一块内存，用来储存你拷贝的不可变内容。

就是说，**对于可变非集合对象的拷贝，copy 和 mutableCopy 都是做的深拷贝。**

### _\_NSTaggedPointerString

Tagged Pointer 是一个能够提升性能、节省内存的有趣的技术。我们知道，程序都使用了指针地址对齐概念。指针地址对齐就是指在分配堆中的内存时往往采用偶数倍或以2为指数倍的内存地址作为地址边界。几乎所有系统架构，包括 Mac OS 和 iOS，都使用了地址对齐概念对象。对于 iOS 和 MAC 来说，指针地址是以16个字节（或16的倍数）为对齐边界的，进一步说，分配的内存地址最后4位永远都是0。

Tagged Pointer 利用了这一现状，它使对象指针中非零位（最后4位）有了特殊的含义。在苹果的64位 Objective-C 实现中，**若对象指针的最低有效位为1\(即奇数\)，则该指针为 Tagged Pointer 。这种指针不通过解引用 isa 来获取其所属类，**而是通过接下来三位的一个类表的索引。该索引是用来查找所属类是采用 Tagged Pointer 的哪个类。剩下的60位则留给类来使用。

Tagged Pointer 有一个简单的应用，那就是 NSNumber 。它使用60位来存储数值。最低位置1。剩下3位为 NSNumber 的标志。这样，就可以存储任何所需内存小于60位的数值。

> 注：以上是在 x86\_64 架构中，**在 iOS ARM64 架构中，是最高4位表示所属类，对于最低位，不同类有不同的意义，比如 NSString 代表的是字符长度 length**，NSNumber 我猜测代表的是数字长度类型。

从外部看，Tagged Pointer很像一个对象。它能够响应消息，因为 objc\_msgSend 可以识别 Tagged Pointer 。假设你调用 integerValue ，它将从那60位中提取数值并返回。这样，每访问一个对象，就省下了一次真正对象的内存分配，省下了一次间接取值的时间。同时引用计数可以是空指令，因为没有内存需要释放。对于常用的类，这将是一个巨大的性能提升。

NSString 也是如此。对于那些所需内存小于60位的字符串，它可以创建一个 Tagged Pointer。所需内存大于60位的则放置在真正的 NSString 对象里。这使得常用的短字符串的性能得到明显的提升。

关于 NSString 中的 Tagged Pointer 编码比较复杂，条件是**长度小于11位，且由 Apple 的代码生成在运行时**，即不是直接定义，而是如上图中 mutableCopy/copy 转换而来，编码详情请见[【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html)

> 在 WWDC2013 中 APPLE 对于它的特点是这样总结的：
>
> 1.  Tagged Pointer 专门用来存储小的对象，例如 NSNumber 和NSDate
> 2.  **Tagged Pointer 指针的值不再是地址了，而是真正的值。所以，实际上它不再是一个对象了，它只是一个披着对象皮的普通变量而已。所以，它的内存并不存储在堆中，也不需要 malloc 和 free 。**跟 \_\_NSCFConstantString 一样拥有非常大的 retainCount ，因为压根儿就不在堆上啊。
> 3.  在内存读取上有着3倍的效率，创建时比以前快106倍。

对 NSString 对象来说，当非字面量的数字，英文字母字符串的长度小于等于11的时候会自动成为 NSTaggedPointerString 类型（赋值为常量除外），如果有中文或其他特殊符号（可能是非 ASCII 字符）存在的话则会直接成为 \_\_NSCFString 类型。

**Tagged Pointer 举例**  
比如我们将上面的字符串改为  
`fC.mString = [NSMutableString stringWithString:@"mStringingin"];`，  
比之前少了1位，只有11位，则输出结果就变为了：

![](/images/2016-10-20-语法-copy-与-mutableCopy（传说中的深浅拷贝）/NSMutableString2.png)

除了拷贝的可变副本（最后一个），其他不可变副本都是 Tagged Pointer ，直接存储的值。

## 集合类对象\(NSArray\)

### 执行代码：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div></pre></td><td class="code"><pre><div class="line">fC.array = [<span class="built_in">NSArray</span> arrayWithObjects:<span class="string">@"hello"</span>,<span class="string">@"world"</span>,<span class="string">@"baby"</span>, <span class="literal">nil</span>];</div><div class="line">fC.arrayCopy = fC.array;  <span class="comment">//浅，指针</span></div><div class="line">fC.arrayMutableCopy = [fC.array mutableCopy]; <span class="comment">//单层深，新地址</span></div><div class="line">        </div><div class="line">fC.mArrayCopy = [fC.array <span class="keyword">copy</span>]; <span class="comment">//浅，指针</span></div><div class="line">fC.mArrayMutableCopy = [fC.array mutableCopy]; <span class="comment">//单层深，新地址</span></div><div class="line">        </div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"\n集合类对象(NSArray)：\noriginal address:    "</span>); AmyLog(fC.array); AmyLog([fC.array objectAtIndex:<span class="number">1</span>]);</div><div class="line">        </div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSArray:    "</span>); AmyLog(fC.arrayCopy); AmyLog([fC.arrayCopy objectAtIndex:<span class="number">1</span>]);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSArray:    "</span>); AmyLog(fC.arrayMutableCopy); AmyLog([fC.arrayMutableCopy objectAtIndex:<span class="number">1</span>]);</div><div class="line"></div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"copy -&gt; NSMutableArray:   "</span>); AmyLog(fC.mArrayCopy); AmyLog([fC.mArrayCopy objectAtIndex:<span class="number">1</span>]);</div><div class="line"><span class="built_in">NSLog</span>(<span class="string">@"mutableCopy -&gt; NSMutableArray:    "</span>); AmyLog(fC.mArrayMutableCopy); AmyLog([fC.mArrayMutableCopy objectAtIndex:<span class="number">1</span>]);</div></pre></td></tr></tbody></table>

### 打印结果：

![](/images/2016-10-20-语法-copy-与-mutableCopy（传说中的深浅拷贝）/NSArray.png)

### 分析

从结果可以发现，对于第一层的指针来说，跟 NSString 是一样的，copy 浅拷贝（复制指针，即指针不变）， mutableCopy 深拷贝（新内存），但是打印数组中的元素，就发现元素的指针并没有变，也就是第二层依然是浅拷贝，因此这就是单层深拷贝了。

### 集合的浅拷贝和完全拷贝

集合的浅拷贝有非常多种方法（上面那种 copy 就是）。当你进行浅拷贝时，会向原始的集合发送retain消息，引用计数加1，同时指针被拷贝到新的集合。

现在让我们看一些浅拷贝的例子：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div></pre></td><td class="code"><pre><div class="line"><span class="built_in">NSArray</span> *shallowCopyArray = [someArray copyWithZone:<span class="literal">nil</span>];   </div><div class="line"><span class="built_in">NSSet</span> *shallowCopySet = [<span class="built_in">NSSet</span> mutableCopyWithZone:<span class="literal">nil</span>];   </div><div class="line"><span class="built_in">NSDictionary</span> *shallowCopyDict = [[<span class="built_in">NSDictionary</span> alloc] initWithDictionary:someDictionary copyItems:<span class="literal">NO</span>];</div></pre></td></tr></tbody></table>

那么如何才能对元素也进行深拷贝呢？

集合的深拷贝有两种方法。可以用 initWithArray:copyItems: 将第二个参数设置为 YES 即可深拷贝，如

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div></pre></td><td class="code"><pre><div class="line"><span class="built_in">NSDictionary</span> shallowCopyDict = [[<span class="built_in">NSDictionary</span> alloc] initWithDictionary:someDictionary copyItems:<span class="literal">YES</span>];</div></pre></td></tr></tbody></table>

如果你用这种方法深拷贝，集合里的每个对象都会收到 copyWithZone: 消息。如果集合里的对象遵循 NSCopying 协议，那么对象就会被深拷贝到新的集合。如果对象没有遵循 NSCopying 协议，而尝试用这种方法进行深拷贝，会在运行时出错。 copyWithZone: 这种拷贝方式只能够提供一层内存拷贝\(one-level-deep copy\)，而非真正的深拷贝。

第二个方法是将集合进行归档\(archive\)，然后解档\(unarchive\)，如：

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div></pre></td><td class="code"><pre><div class="line"><span class="built_in">NSArray</span> *trueDeepCopyArray = [<span class="built_in">NSKeyedUnarchiver</span> unarchiveObjectWithData:[<span class="built_in">NSKeyedArchiver</span> archivedDataWithRootObject:oldArray]];</div></pre></td></tr></tbody></table>

终于搞定这个了！

---

## Reference

\[1\] NSString：内存简述，Copy与Strong关键字　<http://www.jianshu.com/p/0e98f37114e3>  
\[2\] iOS 集合的深复制与浅复制　<https://www.zybuluo.com/MicroCai/note/50592>  
\[3\] 深入理解Tagged Pointe　<http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer/>  
\[4\] 【译】采用Tagged Pointer的字符串　<http://www.cocoachina.com/ios/20150918/13449.html>

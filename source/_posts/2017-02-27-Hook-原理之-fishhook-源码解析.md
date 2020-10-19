---
title: Hook 原理之 fishhook 源码解析
categories: 源码解析
abbrlink: 80b74e12
date: 2017-02-27 00:00:00
---

## 基础知识提要

各种表在 Mach-O 文件中，是位于 Section 数据之后的一些记录数据。下面介绍本文会用到的几个表。

懒加载（lazy load），又叫做延迟加载。在实际需要使用该符号（或资源）的时候，该符号才会通过 dyld 中的 `dyld_stub_binder` 来进行加载。与之相对的是非懒加载（non-lazy load），这些符号在动态链接库绑定的时候，就会被加载。

<!-- more -->

在 Mach-O 中，相对应的就是 \_nl\_symbol\_ptr（非懒加载符号表）和 \_la\_symbol\_ptr（懒加载符号表）。这两个指针表，保存着与字符串表对应的函数指针。

Dynamic Symbol Table\(Indirect Symbols\): 动态符号表是加载动态库时导出的函数表，是符号表的 subset。动态符号表的符号 = 该符号在**原所属表指针**中的偏移量（offset）+ 原所属表在**动态符号表**中的偏移量 + 动态符号表的**基地址**（base）。在动态表中查找到的这个符号的值又等于该符号在 symtab 中的 offset。

Symbol Table（以下简称为 symtab）: 即符号表。每个目标文件都有自己的符号表，记录了符号的映射。在 Mach-O 中，符号表是由结构体 n\_list 构成。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div><div class="line">23</div><div class="line">24</div><div class="line">25</div></pre></td><td class="code"><pre><div class="line"><span class="keyword">struct</span> nlist {</div><div class="line">	<span class="keyword">union</span> {</div><div class="line"><span class="meta">#<span class="meta-keyword">ifndef</span> __LP64__</span></div><div class="line">		<span class="keyword">char</span> *n_name;	<span class="comment">/* for use when in-core */</span></div><div class="line"><span class="meta">#<span class="meta-keyword">endif</span></span></div><div class="line">		<span class="keyword">uint32_t</span> n_strx;	<span class="comment">/* index into the string table */</span></div><div class="line">	} n_un;</div><div class="line">	<span class="keyword">uint8_t</span> n_type;		<span class="comment">/* type flag, see below */</span></div><div class="line">	<span class="keyword">uint8_t</span> n_sect;		<span class="comment">/* section number or NO_SECT */</span></div><div class="line">	<span class="keyword">int16_t</span> n_desc;		<span class="comment">/* see &lt;mach-o/stab.h&gt; */</span></div><div class="line">	<span class="keyword">uint32_t</span> n_value;	<span class="comment">/* value of this symbol (or stab offset) */</span></div><div class="line">};</div><div class="line"></div><div class="line"><span class="comment">/*</span></div><div class="line"> * This is the symbol table entry structure for 64-bit architectures.</div><div class="line"> */</div><div class="line"><span class="keyword">struct</span> nlist_64 {</div><div class="line">    <span class="keyword">union</span> {</div><div class="line">        <span class="keyword">uint32_t</span>  n_strx; <span class="comment">/* index into the string table */</span></div><div class="line">    } n_un;</div><div class="line">    <span class="keyword">uint8_t</span> n_type;        <span class="comment">/* type flag, see below */</span></div><div class="line">    <span class="keyword">uint8_t</span> n_sect;        <span class="comment">/* section number or NO_SECT */</span></div><div class="line">    <span class="keyword">uint16_t</span> n_desc;       <span class="comment">/* see &lt;mach-o/stab.h&gt; */</span></div><div class="line">    <span class="keyword">uint64_t</span> n_value;      <span class="comment">/* value of this symbol (or stab offset) */</span></div><div class="line">};</div></pre></td></tr></tbody></table>

以上为 n\_list 的结构。通过在动态符号表中找的偏移，再加上符号表的基址，就可以找到这个符号的 n\_list，其中 n\_strx 的值代表该字符串在 strtab 中的偏移量（offset）。关于 n\_list 的具体结构解析详见 [nlist-Mach-O文件重定向信息数据结构分析](http://turingh.github.io/2016/05/24/nlist-Mach-O%E6%96%87%E4%BB%B6%E9%87%8D%E5%AE%9A%E5%90%91%E4%BF%A1%E6%81%AF%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%88%86%E6%9E%90/#)

String Table（以下简称为 strtab）: 是放置 Section 名、变量名、符号名的字符串表，字符串末尾自带的 \\0 为分隔符（机器码00）。知道 strtab 的基地址（base），然后加上在 Symbol Table 中找到的该字符串的偏移量（offset）就可以找到这个字符串。

## fishhook 概述

[fishhook](https://github.com/facebook/fishhook) 是 facehook 开源的重绑定 Mach-O 符号的库，用来 hook C 语言函数（即只能重绑定 C 符号）。主要原因在于只针对 C 语言做了符号修饰。

基本思路为：

> 1.  先找到 Mach-O 文件的 Load\_Commands 中的 LC\_SEGMENT\_64\(\_DATA\)，然后找到这条加载指令下的 Section64 Header\(\_nl\_symbol\_ptr\)，以及 Section64 Header\(\_la\_symbol\_ptr\)；
> 2.  其中 Section Header 字段的 `reserved1` 的值即为该 Section 在 Dynamic Symbol Table 中的 offset。然后通过定位到该 Section 的数据，找到目标符号在 Section 中的偏移量，与之前的 offset 相加，即为在动态符号表中的偏移；
> 3.  通过 Indirect Symbols 对应的数值，找到在 symtab 中的偏移，然后取出 n\_list->n\_un->n\_strx 的值；
> 4.  通过这个值找到在 strtab 中的偏移，得到该字符串，进行匹配置换。

## 源码解析

fishhook 的源文件很少，只有一个 .h 头文件和一个 .c 文件，其中 fishhook.h 文件只暴露出了两个函数接口和一个结构体。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div></pre></td><td class="code"><pre><div class="line"><span class="comment">/*</span></div><div class="line"> * A structure representing a particular intended rebinding from a symbol</div><div class="line"> * name to its replacement</div><div class="line"> */</div><div class="line"><span class="keyword">struct</span> rebinding {</div><div class="line">  <span class="keyword">const</span> <span class="keyword">char</span> *name;</div><div class="line">  <span class="keyword">void</span> *replacement;</div><div class="line">  <span class="keyword">void</span> **replaced;</div><div class="line">};</div><div class="line"></div><div class="line"><span class="function"><span class="keyword">int</span> <span class="title">rebind_symbols</span><span class="params">(<span class="keyword">struct</span> rebinding rebindings[], <span class="keyword">size_t</span> rebindings_nel)</span></span>;</div><div class="line"></div><div class="line"><span class="function"><span class="keyword">int</span> <span class="title">rebind_symbols_image</span><span class="params">(<span class="keyword">void</span> *header,</span></span></div><div class="line">                         <span class="keyword">intptr_t</span> slide,</div><div class="line">                         <span class="keyword">struct</span> rebinding rebindings[],</div><div class="line">                         <span class="keyword">size_t</span> rebindings_nel);</div></pre></td></tr></tbody></table>

### rebind\_symbols 函数

以 ReadMe 中的示例为例，先是声明与将要被 hook 的函数签名相同的函数指针，接着自定义了替换后的函数，my\_close、my\_open。然后在 main 函数中调用 rebind\_symbols 函数。

```c
rebind_symbols((struct rebinding[2]){{"close", my_close, (void *)&orig_close}, {"open", my_open, (void *)&orig_open}}, 2);
```

在传递的参数中定义了一个结构体数组，传递了两个 rebinding 结构体，以及数组的个数2。

```c
static struct rebindings_entry *_rebindings_head;
......
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
  int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
  if (retval < 0) {
    return retval;
  }
  // If this was the first call, register callback for image additions (which is also invoked for
  // existing images, otherwise, just run on existing images
  if (!_rebindings_head->next) {
    _dyld_register_func_for_add_image(_rebind_symbols_for_image);
  } else {
    uint32_t c = _dyld_image_count();
    for (uint32_t i = 0; i < c; i++) {
      _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
    }
  }
  return retval;
}
```

在 rebind\_symbols 函数中，首先调用了 prepend\_rebindings 函数，传入了 rebindings\_head 的二级指针， rebind\_symbols 函数参数中的 rebindings 数组，以及数组个数。然后将这个函数的返回值作为整个函数的返回值。

如果这个函数返回值>0，且 \_rebindings\_head->next 的值为空（其具体含义在 prepend\_rebindings 函数中讲），则调用\_dyld\_register\_func\_for\_add\_image 来注册回调函数 \_rebind\_symbols\_for\_image。

在 dyld 加载镜像（即 image，在 Mach-O 中，所有的可执行文件、dylib、Bundle 都是 image）的时候，会执行注册过的回调函数。这一步可以使用 \_dyld\_register\_func\_for\_add\_image 方法来注册自定义的回调函数，传入这个 image 的 mach\_header 和 slide，同时也会为所有已加载的 image 执行回调。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div></pre></td><td class="code"><pre><div class="line"><span class="keyword">extern</span> <span class="keyword">void</span> _dyld_register_func_for_add_image(  </div><div class="line">    <span class="keyword">void</span> (*func)(<span class="keyword">const</span> <span class="keyword">struct</span> mach_header* mh, <span class="keyword">intptr_t</span> vmaddr_slide)</div><div class="line">);</div></pre></td></tr></tbody></table>

如果 \_rebindings\_head->next 的值不为空，则直接调用回调函数。

### prepend\_rebindings 函数

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div><div class="line">23</div><div class="line">24</div><div class="line">25</div><div class="line">26</div></pre></td><td class="code"><pre><div class="line"><span class="keyword">struct</span> rebindings_entry {</div><div class="line">  <span class="keyword">struct</span> rebinding *rebindings;</div><div class="line">  <span class="keyword">size_t</span> rebindings_nel;</div><div class="line">  <span class="keyword">struct</span> rebindings_entry *next;</div><div class="line">};</div><div class="line"></div><div class="line">......</div><div class="line"></div><div class="line"><span class="function"><span class="keyword">static</span> <span class="keyword">int</span> <span class="title">prepend_rebindings</span><span class="params">(<span class="keyword">struct</span> rebindings_entry **rebindings_head,</span></span></div><div class="line">                              <span class="keyword">struct</span> rebinding rebindings[],</div><div class="line">                              <span class="keyword">size_t</span> nel) {</div><div class="line">  <span class="keyword">struct</span> rebindings_entry *new_entry = <span class="built_in">malloc</span>(<span class="keyword">sizeof</span>(<span class="keyword">struct</span> rebindings_entry));</div><div class="line">  <span class="keyword">if</span> (!new_entry) {</div><div class="line">    <span class="keyword">return</span> <span class="number">-1</span>;</div><div class="line">  }</div><div class="line">  new_entry-&gt;rebindings = <span class="built_in">malloc</span>(<span class="keyword">sizeof</span>(<span class="keyword">struct</span> rebinding) * nel);</div><div class="line">  <span class="keyword">if</span> (!new_entry-&gt;rebindings) {</div><div class="line">    <span class="built_in">free</span>(new_entry);</div><div class="line">    <span class="keyword">return</span> <span class="number">-1</span>;</div><div class="line">  }</div><div class="line">  <span class="built_in">memcpy</span>(new_entry-&gt;rebindings, rebindings, <span class="keyword">sizeof</span>(<span class="keyword">struct</span> rebinding) * nel);</div><div class="line">  new_entry-&gt;rebindings_nel = nel;</div><div class="line">  new_entry-&gt;next = *rebindings_head;</div><div class="line">  *rebindings_head = new_entry;</div><div class="line">  <span class="keyword">return</span> <span class="number">0</span>;</div><div class="line">}</div></pre></td></tr></tbody></table>

这里主要是一个将 rebingdings 数组拷贝到 new\_entry 结构体中，并把这个结构体添加到 \_rebings\_head 这个链表首部的操作。首先定义一个 rebindings\_entry 类型的 new\_entry 结构体，并初始化，给 new\_entry 以及 new\_entry->rebindings 分配内存。

然后拷贝传入的参数数组 rebindings 到 new\_entry->rebindings 中。同时给 new\_entry->rebindings\_nel 赋值为数组的个数，将 new\_entry->next 赋值为 \*rebindings\_head 指针，即 \_rebindings\_head 内的数值。最后再使 \_rebindings\_head 与 new\_entry 指向同一个地址。

这里比较容易混淆的是 rebindings\_head 与 \_rebindings\_head。rebind\_symbols 函数调用 prepend\_rebindings 函数时，传入的是 `&_rebindings_head`，也就是结构体指针的地址，是一个二级指针。prepend\_rebindings 函数接收这个参数用的是 `struct rebindings_entry **rebindings_head`，也就是说 \*rebindings\_head 就是 \_rebinding\_head 指针。

![](/images/2017-02-27-源码学习-Hook-原理之-fishhook-源码解析/rebindings.gif)

上面的动图很容易看出，这个链表是如何形成的。回到 rebind\_symbols 函数中的遗留问题，\_rebindings\_head->next 的值为空时，是什么意思？这意味着 rebind\_symbols 函数第一次被调用，因为之后被调用，\_rebindings\_head->next 都指向的是前一个被添加进链表的 new\_entry。也只有在第一次被调用时，才需要注册回调函数，之后都是直接调用即可。

### rebind\_symbols\_for\_image 函数

在 \_rebind\_symbols\_for\_image 中，就执行了一个调用 rebind\_symbols\_for\_image 函数的操作。接下来是比较核心的部分了。

```c
static void rebind_symbols_for_image(struct rebindings_entry *rebindings,
                                     const struct mach_header *header,
                                     intptr_t slide) {
  Dl_info info;
  if (dladdr(header, &info) == 0) {
    return;
  }
  segment_command_t *cur_seg_cmd;
  segment_command_t *linkedit_segment = NULL;
  struct symtab_command* symtab_cmd = NULL;
  struct dysymtab_command* dysymtab_cmd = NULL;
  uintptr_t cur = (uintptr_t)header + sizeof(mach_header_t);
  for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
    cur_seg_cmd = (segment_command_t *)cur;
    if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
      if (strcmp(cur_seg_cmd->segname, SEG_LINKEDIT) == 0) {
        linkedit_segment = cur_seg_cmd;
      }
    } else if (cur_seg_cmd->cmd == LC_SYMTAB) {
      symtab_cmd = (struct symtab_command*)cur_seg_cmd;
    } else if (cur_seg_cmd->cmd == LC_DYSYMTAB) {
      dysymtab_cmd = (struct dysymtab_command*)cur_seg_cmd;
    }
  }
  if (!symtab_cmd || !dysymtab_cmd || !linkedit_segment ||
      !dysymtab_cmd->nindirectsyms) {
    return;
  }
//接下段代码
```

首先定义了4个会被用到的结构体指针。其中 segment\_command\_t 就是LC\_SEGMENT\_64 结构，symtab\_command 是 Section Header 中 LC\_SYMTAB 的结构，dysymtab\_command 是 Section Header 中 LC\_DYSYMTAB 的结构。（有关 Mach-O 文件的结构可以参考我之前的文章 [解读 Mach-O 文件格式](https://harpersu00.github.io/eeb03f45.html)）

接下来跳过 Mach-O 的 Header 结构，开始遍历 Load Commands。通过 Header->ncmds，以及 Segment->cmdsize 来控制循环。通过遍历，找到 LC\_SEGMENT\_64\(\_LINKEDIT\)，赋值给 linkedit\_segment，然后给 symtab\_cmd 和 dysymtab\_cmd 赋值。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div></pre></td><td class="code"><pre><div class="line"><span class="comment">//接上段代码</span></div><div class="line"><span class="comment">// Find base symbol/string table addresses</span></div><div class="line">  <span class="keyword">uintptr_t</span> linkedit_base = (<span class="keyword">uintptr_t</span>)slide + linkedit_segment-&gt;vmaddr - linkedit_segment-&gt;fileoff;</div><div class="line">  <span class="keyword">nlist_t</span> *symtab = (<span class="keyword">nlist_t</span> *)(linkedit_base + symtab_cmd-&gt;symoff);</div><div class="line">  <span class="keyword">char</span> *strtab = (<span class="keyword">char</span> *)(linkedit_base + symtab_cmd-&gt;stroff);</div><div class="line"></div><div class="line">  <span class="comment">// Get indirect symbol table (array of uint32_t indices into symbol table)</span></div><div class="line">  <span class="keyword">uint32_t</span> *indirect_symtab = (<span class="keyword">uint32_t</span> *)(linkedit_base + dysymtab_cmd-&gt;indirectsymoff);</div><div class="line"><span class="comment">//接下段代码</span></div></pre></td></tr></tbody></table>

通过找到的 \_LINKEDIT 段和传入的参数 slide 来计算 base 地址。也就是这个 Mach-O 文件在 ASLR 偏移后的首地址（因为这里用到的这些表都是属于 \_LINKEDIT 段的）。 base = vmaddr \- fileoffset + slide。

然后 base + symtab 段的 Symbol Table Offset（该表在文件中的偏移） = symtab 的首地址（该表在内存中的偏移），base + symtab 段的 String Table Offset = strtab 的首地址。base + DYSYTAB 段的 IndSym Table Offset = 动态符号表的首地址。

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div></pre></td><td class="code"><pre><div class="line"><span class="comment">//接上段代码</span></div><div class="line">cur = (<span class="keyword">uintptr_t</span>)header + <span class="keyword">sizeof</span>(<span class="keyword">mach_header_t</span>);</div><div class="line">  <span class="keyword">for</span> (uint i = <span class="number">0</span>; i &lt; header-&gt;ncmds; i++, cur += cur_seg_cmd-&gt;cmdsize) {</div><div class="line">    cur_seg_cmd = (<span class="keyword">segment_command_t</span> *)cur;</div><div class="line">    <span class="keyword">if</span> (cur_seg_cmd-&gt;cmd == LC_SEGMENT_ARCH_DEPENDENT) {</div><div class="line">      <span class="keyword">if</span> (<span class="built_in">strcmp</span>(cur_seg_cmd-&gt;segname, SEG_DATA) != <span class="number">0</span> &amp;&amp;</div><div class="line">          <span class="built_in">strcmp</span>(cur_seg_cmd-&gt;segname, SEG_DATA_CONST) != <span class="number">0</span>) {</div><div class="line">        <span class="keyword">continue</span>;</div><div class="line">      }</div><div class="line">      <span class="keyword">for</span> (uint j = <span class="number">0</span>; j &lt; cur_seg_cmd-&gt;nsects; j++) {</div><div class="line">        <span class="keyword">section_t</span> *sect =</div><div class="line">          (<span class="keyword">section_t</span> *)(cur + <span class="keyword">sizeof</span>(<span class="keyword">segment_command_t</span>)) + j;</div><div class="line">        <span class="keyword">if</span> ((sect-&gt;flags &amp; SECTION_TYPE) == S_LAZY_SYMBOL_POINTERS) {</div><div class="line">          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);</div><div class="line">        }</div><div class="line">        <span class="keyword">if</span> ((sect-&gt;flags &amp; SECTION_TYPE) == S_NON_LAZY_SYMBOL_POINTERS) {</div><div class="line">          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);</div><div class="line">        }</div><div class="line">      }</div><div class="line">    }</div><div class="line">  }</div><div class="line">}</div></pre></td></tr></tbody></table>

这是一个嵌套循环，外层循环依旧是遍历 Load Commands，内循环则是遍历 LC\_SEGMENT\_64\(\_DATA\) 段内的 Section Header，通过 Section->flags \& SECTION\_TYPE 来寻找 \_nl\_symbol\_ptr 和 \_la\_symbol\_ptr。找到后调用 perform\_rebinding\_with\_section 函数。

> 在循环内的 if 语句嵌套，一般最多用两层，太多层会显得代码冗杂，且可读性较差，容易出错。那两层以上怎么办呢？fishhook 给了我们一个很好的示范。用 if 语句作非判断，然后加上 continue 跳出本次循环。详见上面的代码。

### perform\_rebindin\_with\_section 函数

<table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div><div class="line">23</div><div class="line">24</div><div class="line">25</div><div class="line">26</div><div class="line">27</div><div class="line">28</div><div class="line">29</div><div class="line">30</div><div class="line">31</div><div class="line">32</div><div class="line">33</div><div class="line">34</div><div class="line">35</div><div class="line">36</div></pre></td><td class="code"><pre><div class="line"><span class="function"><span class="keyword">static</span> <span class="keyword">void</span> <span class="title">perform_rebinding_with_section</span><span class="params">(<span class="keyword">struct</span> rebindings_entry *rebindings,</span></span></div><div class="line">                                           <span class="keyword">section_t</span> *section,</div><div class="line">                                           <span class="keyword">intptr_t</span> slide,</div><div class="line">                                           <span class="keyword">nlist_t</span> *symtab,</div><div class="line">                                           <span class="keyword">char</span> *strtab,</div><div class="line">                                           <span class="keyword">uint32_t</span> *indirect_symtab) {</div><div class="line">  <span class="keyword">uint32_t</span> *indirect_symbol_indices = indirect_symtab + section-&gt;reserved1;</div><div class="line">  <span class="keyword">void</span> **indirect_symbol_bindings = (<span class="keyword">void</span> **)((<span class="keyword">uintptr_t</span>)slide + section-&gt;addr);</div><div class="line">  <span class="keyword">for</span> (uint i = <span class="number">0</span>; i &lt; section-&gt;size / <span class="keyword">sizeof</span>(<span class="keyword">void</span> *); i++) {</div><div class="line">    <span class="keyword">uint32_t</span> symtab_index = indirect_symbol_indices[i];</div><div class="line">    <span class="keyword">if</span> (symtab_index == INDIRECT_SYMBOL_ABS || symtab_index == INDIRECT_SYMBOL_LOCAL ||</div><div class="line">        symtab_index == (INDIRECT_SYMBOL_LOCAL   | INDIRECT_SYMBOL_ABS)) {</div><div class="line">      <span class="keyword">continue</span>;</div><div class="line">    }</div><div class="line">    <span class="keyword">uint32_t</span> strtab_offset = symtab[symtab_index].n_un.n_strx;</div><div class="line">    <span class="keyword">char</span> *symbol_name = strtab + strtab_offset;</div><div class="line">    <span class="keyword">if</span> (strnlen(symbol_name, <span class="number">2</span>) &lt; <span class="number">2</span>) {</div><div class="line">      <span class="keyword">continue</span>;</div><div class="line">    }</div><div class="line"> <span class="keyword">struct</span> rebindings_entry *cur = rebindings;</div><div class="line">    <span class="keyword">while</span> (cur) {</div><div class="line">      <span class="keyword">for</span> (uint j = <span class="number">0</span>; j &lt; cur-&gt;rebindings_nel; j++) {</div><div class="line">        <span class="keyword">if</span> (<span class="built_in">strcmp</span>(&amp;symbol_name[<span class="number">1</span>], cur-&gt;rebindings[j].name) == <span class="number">0</span>) {</div><div class="line">          <span class="keyword">if</span> (cur-&gt;rebindings[j].replaced != <span class="literal">NULL</span> &amp;&amp;</div><div class="line">              indirect_symbol_bindings[i] != cur-&gt;rebindings[j].replacement) {</div><div class="line">            *(cur-&gt;rebindings[j].replaced) = indirect_symbol_bindings[i];</div><div class="line">          }</div><div class="line">          indirect_symbol_bindings[i] = cur-&gt;rebindings[j].replacement;</div><div class="line">          <span class="keyword">goto</span> symbol_loop;</div><div class="line">        }</div><div class="line">      }</div><div class="line">      cur = cur-&gt;next;</div><div class="line">    }</div><div class="line">  symbol_loop:;</div><div class="line">  }</div><div class="line">}</div></pre></td></tr></tbody></table>

这里需要注意的是指针的加法。比如 `uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1;` 这句代码中，三个变量均为 uint32\_t 格式，即4个字节，那么 indirect\_symtab 指针实际应该加上 （reserved1 的值 \* 4）个字节。以 \_nl\_symbol\_ptr 为例：

![](/images/2017-02-27-源码学习-Hook-原理之-fishhook-源码解析/dysymtab.png)

reserved1（也就是上图 MachOView 中显示的 Indirect Sym Index）为25，那么在动态符号表中 \_nl\_symbol\_ptr 所在的首地址应该是（先不考虑 slide）： Dynamic Symbol Table 的首地址 + \(reserved1 \* 4\) = 0x100005F30 + 0x64 = 0x100005F94。

slide + section->addr 为 \_nl\_symbol\_ptr section 数据所在的地址。然后遍历 dysymtab 中从 \_nl\_symbol\_ptr 开始的符号，取得 Symbol 数据，如果为 INDIRECT\_SYMBOL\_ABS\(即懒加载符号指针结束的地方\)等，则跳出本次循环。否则将取得的 Symbol 数据作为 symtab 中的 offset。

![](/images/2017-02-27-源码学习-Hook-原理之-fishhook-源码解析/symtab.png)

如上图所示，\_dyld\_stub\_binder 符号（也是这个程序中的 \_nl\_symbol\_ptr 的首个符号）在 dysymtab 中的 Symbol 数据为0xA1，那么在对应的 symtab 中，它的地址应为 symtab 首地址 + 0xA1 \* 16 = 0x10005510 + 161 \* 16 = 0x10005F20。\(16是 n\_list 结构共16个字节\)

Symbol Table 中的 n\_strx（之前提到的 n\_list 结构）即为 strtab 中的 index。

![](/images/2017-02-27-源码学习-Hook-原理之-fishhook-源码解析/strtab.png)

最后就是匹配替换了。比较字符串表中的字符串与 rebindings 数组中的 name 字段，匹配成功后，将 \_nl\_symbol\_ptr 或 \_la\_symbol\_ptr 这两个 Section 的指针表中对应的函数指针（`indirect_symbol_bindings[i]`）赋值给 rebindings 数组中的 replaced 字段，然后用数组中的 replacement 字段（也就是自定义的 my\_open 或 my\_close 函数的指针）覆盖原来的函数指针。

> 这里使用了 goto 来跳出双重循环，值得参考。

* * *

## Reference

\[1\] 动态修改 C 语言函数的实现　<http://draveness.me/fishhook/>  
\[2\] 趣探 Mach-O：FishHook 解析　<http://www.open-open.com/lib/view/open1487057519754.html>  
\[3\] 编译体系漫游　<http://www.tuicool.com/articles/uI7Bria>

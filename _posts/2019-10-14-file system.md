---
layout:     post
title:      "操作系统（五）：文件系统（二）"
subtitle:   "Minix文件系统之数据块缓存"
date:       2019-10-14
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 操作系统
      文件系统
---





### 前言
---
> 由于处理器与硬盘之间数据操作的速度具有显著差异，并且硬盘上数据的分布一般具有[局部性原理](https://baike.baidu.com/item/%E5%B1%80%E9%83%A8%E6%80%A7%E5%8E%9F%E7%90%86/3334556?fr=aladdin)特征，所以计算机科学家们想出了在处理器和硬盘之间通过额外增加一个空间不大但高速的缓存结构来提高处理器执行效率，这也就是缓存出现的背景。缓存在当今计算机系统中的应用案例比比皆是，文件系统当然也不例外，本文作为文件系统系列文章第二篇主要来对块设备文件结构层次中的**缓存**（Cache）部分进行总结归纳（后文所述文件系统未提及具体名称默认为 Minix 1.0） 。                                                                                                                       



### 1 缓存在文件系统中的应用？

---
&emsp;&emsp;在详细讲述缓存在文件系统中所扮演角色前，我们还是先再来回顾一下文件系统的架构图:

<div align="center"><img src="/img/in-post/content/os/file-system/file-view-cache.png" width="80%"/><b>Figure 1：Unified file view</b></div>

&emsp;&emsp;由于块设备的结构较为典型，上图仅绘制了块设备的缓存结构，但字符设备的数据传输过程也是有缓存的，每一种具体外设都有各自的缓存队列，有兴趣的读者可以查找相关资料进行学习。由上图可知，块设备文件系统结构中缓存处于中间层，在数据读写访问过程中起着承上启下的作用。处理器对块设备数据读写访问基本过程如下：

1. 处理器将所需访问的块设备文件结构中的设备号 dev和块号 block作为参数传递给缓存结构控制器发起对相应块数据的请求。
2. 缓存控制器根据参数信息检索缓存表，如果命中，则直接返回，将该数据块传递发送给处理器进行操作。
3. 否则，先在缓存空闲表中查找一块空闲块用于存放所请求的数据块*（Linux-0.11中缓存块与硬盘数据块大小相等都为 1024字节）*。
4. 在将基本的 dev和 block信息赋给空缓存块结构后。再将该结构指针作为参数调用设备驱动控制程序接口`ll_rw_block()`以请求读取硬盘中的目标块。驱动程序读取磁盘数据块的详细故事参见上一篇文章[Minix文件系统之设备驱动]()所述。***(记得更新链接)***

&emsp;&emsp;处理器通过文件结构信息请求缓存读取数据的过程大致如下图所示，其中的 `bread`和`getblk`是获取缓存块的相关接口。

<div align="center"><img src="/img/in-post/content/os/file-system/request-cache.png" width="60%"/><b>Figure 2：Workflow</b></div>

### 2 缓存实现
---
&emsp;&emsp;通过上文，我们已经知晓缓存在文件系统中的具体应用。但仅仅通过上述内容的叙述显然无法真正明白和体会到缓存在文件系统中的价值与美感。接下来，通过 Linux-0.11中相关源代码来进一步认识文件系统中的缓存。

#### 2.1 缓存数据结构

&emsp;&emsp;整个缓存区是一块固定大小的内存空间，内部被划分成一个个 1024字节大小的缓冲块，与块设备上磁盘逻辑块大小相当。缓冲采用 **Hash表**和包含所有缓冲块的**链表**进行管理操作。其上存储着两种结构信息——**缓存块数据**和**缓存块头结构描述符**。在缓存区初始化过程中，初始化程序从整个缓存区两端开始，分别同时设置缓冲头结构和划分出对应的缓冲块，见下图 Figure 3所示。缓冲区高端被划分为一个个 1024字节的缓存块，低端则分别建立起对应各缓存块的缓存头结构`buffer_head`，该头结构用于描述对应缓存块的属性，并用于构造缓存中的缓存块链表。

&emsp;&emsp;所有缓存块的`buffer_head`被链接成一个双向链表结构，见下图 Figure 4所示。图中 `free_list`指针是该链表的头指针，指向空闲块链表中第一个**“最近最少使用的块”**。而该缓冲块的反向指针`b_prev_free`则指向缓存块链表中的最后一个缓冲块，即最近刚使用过的缓存块。当需要置换一块数据出缓存区时就可从头指针所指块开始向后找出一个最近最少使用并可释放的缓存块（未被锁定）即可。这就是教科书中经典的 **LRU**（Least Recently Used）页面置换算法。

```c
/* linux-0.11/include/linux/fs.h */
struct buffer_head {
	char * b_data;			/* pointer to data block (1024 bytes) */
	unsigned long b_blocknr;	/* block number */
	unsigned short b_dev;		/* device (0 = free) */
	unsigned char b_uptodate;   /* 0-not updated,1-updated*/
	unsigned char b_dirt;		/* 0-clean,1-dirty */
	unsigned char b_count;		/* users using this block */
	unsigned char b_lock;		/* 0 - ok, 1 -locked */
	struct task_struct * b_wait;
	struct buffer_head * b_prev;
	struct buffer_head * b_next;
	struct buffer_head * b_prev_free;
	struct buffer_head * b_next_free;
};
```

&emsp;&emsp;上述结构中，`b_data`字段是一个指向所对应缓存块数据的指针，`b_blocknr`和 `b_dev`分别是该头指针所对应的数据块所属设备的**设备号**和其**逻辑块号**，作为向驱动程序发送读写请求的参数，以上两个属性可唯一标识一个数据块。剩下的 `b_uptodate`和 `b_dirt`分别表示数据块更新是否有效和数据块数据是否已脏（即与硬盘对应块数据不一致）。`b_count`表示使用该块的进程数，`b_lock`表示块是否被锁定，其作用是为了防止当驱动程序对缓存块读写数据，占用缓存块的进程处于睡眠时，其他进程使用该缓存块。`b_wait`指向等待该缓冲区解锁的任务。最后，`b_prev`和 `b_next`分别指向该块在缓存 Hash表中的前一个和后一个块，`b_prev_free`和 `b_next_free`分别指向该块在存储空闲表中前一块和后一块缓存，如Figure 4所示。

<div align="center"><img src="/img/in-post/content/os/file-system/initial-cache.png" width="80%"/><b>Figure 3：Initial cache</b></div>

<div align="center"><img src="/img/in-post/content/os/file-system/cacheblock-structure.png" width="90%"/><b>Figure 4：Cacheblock structure</b></div>

&emsp;&emsp;尽管基于链表实现的缓存块空闲表方便查找空闲块，但判断缓存是否命中却非常耗时。因此，为了高效在缓存中检索数据块，文件系统在缓存中引入了Hash表结构以提高块检索效率。Linux-0.11中缓存结构 Hash表项数目为 307项，具体Hash函数为：***（设备号^逻辑块号）Mod 307***。其大致结构见下图：

<div align="center"><img src="/img/in-post/content/os/file-system/hashtable-structure.png" width="85%"/><b>Figure 5：Hashtable structure</b></div>

&emsp;&emsp;上图中双箭头表示散列在同一 Hash表项中缓存块头结构之间的双向链表指针。虚线表示缓冲区中所有空闲缓存块组成的一个双向循环链表，即上图Figure 4结构。

#### 2.2 缓存块搜索

&emsp;&emsp;上文已经对文件系统缓存结构进行了较为详细的描述。最后，将对处理器如何根据块设备号和逻辑块号通过缓存区读写数据块过程再进行归纳。Linux-0.11中缓存供处理器调用请求缓存块的接口函数为：`struct buffer_head * getblk(int dev,int block)`。对于该函数的详细调用过程本文不再赘述，有兴趣的读者可参考[Linux内核完全注解](https://book.douban.com/subject/1231236/) P535页内容和 `getblk`函数源码实现等资料，本文仅将其执行逻辑流程图进行展示：

<div align="center"><img src="/img/in-post/content/os/file-system/getblk-process.png" width="75%"/><b>Figure 6：getblk process</b></div>



### 3 小结
---
- 本文作为文件系统系列文章的第二篇，在[第一篇:设备驱动程序]()的基础上对块设备文件系统中的缓存实现所涉及的数据结构和算法等内容进行了一个较为详细的阐述和小结。文件系统中 Layer 2请求 Layer 3驱动程序读写硬盘数据块的参数为 `buffer_header`指针，Layer 1请求 Layer 2数据块缓存的参数为块设备号 dev以及逻辑块号 block。下一篇文章我们将来看看文件系统是如何根据所操作文件的文件名等标识符提取出其中的 dev和 block等信息传递给缓存，通过文件抽象实现操纵设备读写。

  ​




### 参考资料
---
- [Linux内核完全注释，赵炯](https://book.douban.com/subject/1231236/)
- [哈尔滨工业大学操作系统课程，李治军](https://www.bilibili.com/video/av17036347?from=search&seid=11186295937821986776)


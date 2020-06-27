---
layout:     post
title:      "操作系统（四）：文件系统（一）"
subtitle:   "Minix文件系统之设备驱动"
date:       2019-10-11
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 操作系统
      文件系统
---





### 前言
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2019/10/11/file-system/)
>
> 在通过前文对操作系统基础知识的学习后，接下来，便可开始深入了解其核心模块的设计实现了。Linux-0.11作为了一个早期版本的 Linux内核，麻雀虽小，五脏俱全，已经具备了操作系统内核最核心的两大功能——**CPU管理**和**文件系统**。本文将以 Linux-0.11中 Minix 1.0文件系统为叙述对象对操作系统中文件系统模块进行一个较为深入地梳理和总结。由于该部分内容较多计划分三篇文章从其三个部分来分别讲述，本文作为其中的第一篇文章将从其第一部分——设备驱动展开来认识文件系统（后文所述文件系统未提及具体名称默认为 Minix 1.0）。                                                                                                                           



### 1 文件系统有什么用？
---
&emsp;&emsp;初看标题，读者可能会觉得这不是废话，文件系统不就是用来管理文件吗？对于没学过操作系统的人来说，这个答案确实就差不多了。但对于计算机学习者来说，如此粗浅地理解显然是不够的。首先，来看对于计算机**文件系统**一词维基百科给出的解释如下：

> 计算机的**文件系统**是一种存储和组织计算机数据的方法，它使得对其访问和查找变得容易，文件系统使用**文件**和**树形目录**的抽象逻辑概念代替了硬盘和光盘等物理设备使用数据块的概念，用户使用文件系统来保存数据不必关心数据实际保存在硬盘（或者光盘）的地址为多少的数据块上，只需要知道该文件的所属目录和文件名。

&emsp;&emsp;将上述内容进行总结，文件系统就是在物理存储设备上提供了一层**抽象**（Abstraction）。通过屏蔽硬件细节，让用户更加简单地仅通过文件名等形式便可实现对数据的操作和管理。

&emsp;&emsp;在操作系统中，文件系统通过一个**统一的文件视图**来对所有外设进行管理。一图胜千言，通过一幅图来初步认识文件系统的结构：

<div align="center"><img src="/img/in-post/content/os/file-system/unified-file-view.png" width="80%"/><b>Figure 1：Unified file view</b></div>

&emsp;&emsp;外设作为用户与计算机交互的入口，作为现代计算机组成中核心模块，扮演着非常重要的角色。一般的操作系统将外设粗略地分成两种类型：**块设备**（block device）和**字符设备**（character device）。块设备是一种可以以固定大小的数据块为单位进行寻址和访问的设备，如上图右侧的硬盘和软盘设备等。字符设备是一种以字符流作为操作对象的设备，不能进行寻址操作。如上图左侧的显示器，键盘和打印机等。相比于块设备而言，流设备的操作流程较简单，文本不做介绍，有兴趣的读者可以参考[《Linux内核完全注释》](https://book.douban.com/subject/1231236/)一书中第十章对流设备相关内容。本系列文章将按照上图所绘结构，以自底向上的顺序来叙述 Minix 1.0文件系统管理块设备的具体实现，本篇文章将通过对上图中 Layer 3中块设备与系统交互模块——设备驱动进行阐述来认识文件系统如何从源头进行数据的读写管理。



### 2 文件系统之设备驱动
---
#### 2.1 请求结构

&emsp;&emsp;由计算机组成原理知识可知，内存属于易失型存储器，断电后就会丢失其上所有数据。而硬盘则为持久性存储设备，只要不人为地删除，其上的数据一般都可长久保存。所以通常程序和文件等数据都存放在硬盘上，当系统开机后需要使用某个文件或程序时再动态将其加载入内存由处理器进行执行。本节将讲述的驱动程序将展示计算机文件系统如何从数据源头对数据块进行读写处理。

&emsp;&emsp;Linux-0.11的 Minix文件系统从设备读取数据的过程为，Layer 2中的缓冲管理程序通过调用 Layer 3中提供的低级块读写函数 `ll_rw_block()`，向相应的块设备驱动程序发出一个读/写数据块的操作请求。然后，该函数为此创建一个**请求结构项**，并插入到相应设备的请求队列中。为了提高读写磁盘的效率，减少磁头移动的距离，在插入请求项时会通过**电梯算法**对同一种设备的请求项按照一定的规则进行排序。

&emsp;&emsp;接下来将结合示意图以及 Linux-0.11内核相关具体代码对上述过程进行详细说明。首先，为了接收不同块设备的请求，Linux-0.11通过一张块设备表 `lbk_dev[]`来对不同设备的请求项分别进行管理。每种块设备都在表中占一项，块设备结构表表项结构如下：

```c++
/* linux-0.11/kernel/blk_drv/blk.h*/
struct blk_dev_struct {
	void (*request_fn)(void);         //Function pointer for request item operation
	struct request * current_request; //Current request item pointer
};

//Block device table(array，NR_BLK_DEV=7)
extern struct blk_dev_struct blk_dev[NR_BLK_DEV];
```

&emsp;&emsp;设备结构表结构体中，第一个字段为函数指针，其指向对应具体块设备驱动程序中的执行函数，用于调用具体块设备驱动程序处理相应块设备链表中的请求项。例如：对于硬盘设备，它为 `do_hd_request()`，对于软盘设备，它为 `do_floppy_request()`。第二个字段是当前请求项结构指针，用于指向本块设备请求项队列的表头，初始化时都被设置成 NULL。在 Linux-0.11中，默认的块设备有三种，它们分别对应的请求信息如下：

| 主设备号 |  类型  |       说明       |     请求项操作函数     |
| :--: | :--: | :------------: | :-------------: |
|  1   | 块/字符 | ram, 内存设备，虚拟盘等 | do_rd_request() |
|  2   |  块   |    fd, 软驱设备    | do_fd_request() |
|  3   |  块   |    hd, 硬盘设备    | do_hd_request() |

&emsp;&emsp;当内核发出一个块设备读写请求，`ll_rw_block()`函数即根据其参数中指明的命令(读/写)和设备号等信息，构造一个对应的请求项，然后再插入到块设备表 `blk_dev[]`中对应设备项所指向的请求队列中。块设备请求项结构如下：

```c
/* linux-0.11/kernel/blk_drv/blk.h */
struct request {
	int dev;		       /* -1 if no request */
	int cmd;		       /* READ or WRITE */
	int errors;                    /* the time of execution error*/
	unsigned long sector           /* starting sector*/
	unsigned long nr_sectors;      /* the sum number of sector need reading/writing this*/
	char * buffer;                 /* buffer to storage block that interaction with harddisk*/
	struct task_struct * waiting;  /* the process is waiting for the request*/
	struct buffer_head * bh;       /* pointer of buffer block*/
	struct request * next;         /* the next request in linklist*/
};
extern struct request request[NR_REQUEST];
```

&emsp;&emsp;每个块设备的当前请求(`current_request`) 指针与请求项数组中该设备的请求项链表共同构成了块设备的请求队列。项与项之间利用字段 next指针形成链表。因此块设备项和相应的请求队列的数组形成如下图所示结构，请求项采用数组加链表结构的好处有两点：一是利用请求项的数组结构在搜索空闲请求块时可进行循环操作，速度快。二是为了方便电梯算法将新请求项插入到请求项链表上合适的位置中。

<div align="center"><img src="/img/in-post/content/os/file-system/request-structure.png" width="90%"/><b>Figure 2：Request structure</b></div>

#### 2.2 电梯算法

&emsp;&emsp;上文已经提及，在块请求项插入请求队列过程中会通过电梯算法根据磁头移动最少等原则将其放到最合适的地方，以便提高磁盘整体的读写效率。接下来我们从具体的代码来了解其实现过程。

```c++
/* linux-0.11/kernel/blk_drv/ll_rw_blk.c */
static void add_request(struct blk_dev_struct * dev, struct request * req)
{
	struct request * tmp;

	req->next = NULL;
	cli();
	if (req->bh)
		req->bh->b_dirt = 0;
	//if dev is null, add request to header directly.
	if (!(tmp = dev->current_request)) {
		dev->current_request = req;
		sti();
		(dev->request_fn)();
		return;
	}
	//if not, utilize the elevator algorithm to insert it into REQUEST list.
	for ( ; tmp->next ; tmp=tmp->next)
		if ((IN_ORDER(tmp,req) || 
		    !IN_ORDER(tmp,tmp->next)) &&
		    IN_ORDER(req,tmp->next))
			break;
	req->next=tmp->next;
	tmp->next=req;
	sti();
}

/*
 * This is used in the elevator algorithm: Note that
 * reads always go before writes. This is natural: reads
 * are much more time-critical than writes.
 */
#define IN_ORDER(s1,s2) \
((s1)->cmd<(s2)->cmd || ((s1)->cmd==(s2)->cmd && \
((s1)->dev < (s2)->dev || ((s1)->dev == (s2)->dev && \
(s1)->sector < (s2)->sector))))
```

&emsp;&emsp;函数实现了将新请求项 request插入到请求表中的业务逻辑。首先，注意函数第一个参数 `blk_dev_struct`指针指向调用前根据设备号获得的特定块设备表项。第7行是通过开关中断保护这段临界区（Linux-0.11运行在单CPU上，可使用开关中断实现临界区保护），对应第13和25行有相应的解锁操作。该处通过开关中断来避免多线程下的插入异常。再看到第 10行，当该设备请求表还没有请求项时，直接将该请求项插入其中然后通过13行调用对应的设备驱动程序开始处理请求项链表上的请求。当设备请求表头指针不为空时，通过电梯算法将 request插入请求链表中合适的位置上。电梯算法实现定义在 `IN_ORDER`宏函数中。

#### 2.3 块设备读写流程

&emsp;&emsp;通过上述内容，我们已经了解了块设备请求项的基本组织形式等细节。接下来，就通过磁盘（HardDisk）驱动程序代码来看看磁盘驱动程序如何处理其对应块设备请求队列中的请求项。在讲解代码前，先再把目光投到上文 `add_request()`函数第13行找到对应设备驱动程序执行的开始 `(dev->request_fn)()`，该句通过函数指针的方式执行对应设备的驱动程序入口函数，硬盘是 do_hd_request()，软盘是 do_fd_request()，虚拟盘是 do_rd_request()等。本节我们仅通过硬盘驱动程序为案例来介绍块设备读写块过程，其他设备思路也大致相同有兴趣可阅读相关源码进行学习。硬盘驱动程序中核心源码如下，为了便于说明，仅保留了核心执行流程代码，删掉了一些额外细节：

```c
/*linux-0.11/kernel/hd.c */
void do_hd_request(void)
{
	int i,r = 0;
	unsigned int block,dev;
	unsigned int sec,head,cyl;
	unsigned int nsect;

	INIT_REQUEST;
	dev = MINOR(CURRENT->dev);
	block = CURRENT->sector;
	... /* Omit some sector checks*/
	block += hd[dev].start_sect;
	dev /= 5;
	__asm__("divl %4":"=a" (block),"=d" (sec):"0" (block),"1" (0),
		"r" (hd_info[dev].sect));
	__asm__("divl %4":"=a" (cyl),"=d" (head):"0" (block),"1" (0),
		"r" (hd_info[dev].head));
	sec++;
	nsect = CURRENT->nr_sectors;
	... /* Omit some Hard disk hardware flag checks*/
	if (CURRENT->cmd == WRITE) {
		hd_out(dev,nsect,sec,head,cyl,WIN_WRITE,&write_intr);
	    .../* Omit some Hard disk hardware checks*/
	} else if (CURRENT->cmd == READ) {
		hd_out(dev,nsect,sec,head,cyl,WIN_READ,&read_intr);
	} else
		panic("unknown hd-command");
}

/*linux-0.11/kernel/hd.c, sent WRITE/READ command to control of harddisk*/
static void hd_out(unsigned int drive,unsigned int nsect,unsigned int sect,
		unsigned int head,unsigned int cyl,unsigned int cmd,
		void (*intr_addr)(void))
{
	register int port asm("dx");

	... /* Omit some block checks*/
	do_hd = intr_addr;
	outb_p(hd_info[drive].ctl,HD_CMD);
	port=HD_DATA;
	outb_p(hd_info[drive].wpcom>>2,++port);
	outb_p(nsect,++port);
	outb_p(sect,++port);
	outb_p(cyl,++port);
	outb_p(cyl>>8,++port);
	outb_p(0xA0|(drive<<4)|head,++port);
	outb(cmd,++port);
}

/*linux-0.11/kernel/hd.c, READ command*/
static void read_intr(void)
{
	.../* Omit some checks*/
	port_read(HD_DATA,CURRENT->buffer,256);
	CURRENT->errors = 0;
	CURRENT->buffer += 512;
	CURRENT->sector++;
	if (--CURRENT->nr_sectors) {
		do_hd = &read_intr;
		return;
	}
	end_request(1);
	do_hd_request();
}

/*linux-0.11/kernel/hd.c, WRITE command*/
static void write_intr(void)
{
	.../* Omit some checks*/
	if (--CURRENT->nr_sectors) {
		CURRENT->sector++;
		CURRENT->buffer += 512;
		do_hd = &write_intr;
		port_write(HD_DATA,CURRENT->buffer,256);
		return;
	}
	end_request(1);
	do_hd_request();
}

/*linux-0.11/kernel/blk.h, */
static inline void end_request(int uptodate)
{
	DEVICE_OFF(CURRENT->dev);
	if (CURRENT->bh) {
		CURRENT->bh->b_uptodate = uptodate;
		unlock_buffer(CURRENT->bh);
	}
	if (!uptodate) {
		printk(DEVICE_NAME " I/O error\n\r");
		printk("dev %04x, block %d\n\r",CURRENT->dev,
			CURRENT->bh->b_blocknr);
	}
	wake_up(&CURRENT->waiting);
	wake_up(&wait_for_request);
	CURRENT->dev = -1;
	CURRENT = CURRENT->next;
}
```

&emsp;&emsp;首先，我们从头开始看硬盘驱动程序的执行入口函数 `do_hd_request()`，第9行的 `INIT_REQUEST`是一段宏调用用来检查请求项的合法性，若请求队列中已经没有请求项则退出。然后，第10~21行代码是将请求项参数解析成硬盘中具体扇区等信息，为磁盘控制器在特定扇区，磁道和柱面上进行读写数据提供详细信息。第22~26行是该函数的核心功能，根据请求命令类型（读/写），分别传递不同参数调用 `hd_out()`函数执行对应的逻辑。

&emsp;&emsp;`hd_out()`函数作为驱动程序与硬盘控制器的交互函数，承担了发送读/写命令给硬盘控制器的任务。首先看到第36行，定义局部寄存器变量并存放在指定通用寄存器 dx中，该处应结合第 41行来看，置 dx为数据寄存器端口（0xlf0）。然后通过第 42~48行参数的传递和 port变量指针的移动将读/写硬盘所需的磁盘详细信息设置到 **0x1f1~0x1f7**这 7个端口上。另外，第39行的 `do_hd`所指函数会在硬盘控制器发出中断后的中断处理程序中被调用，将参数 `intr_addr`函数指针所指的 `read_intr()/write_intr()`函数赋给其，当硬盘完成了一次任务向 CPU发出中断时，在对应的中断处理程序中，调用本次请求的 `do_hd`指针所指向的任务函数进行执行。最终，硬盘控制器通过特定端口的参数获取读/写硬盘的具体信息，便可开始执行数据块的读/写请求。

&emsp;&emsp;以上就是请求项执行的基本过程，接下来再对其中的执行函数 read_intr()和write_intr()进行一个较详细的介绍，通过阅读其具体实现便可理解一个设备请求队列中的任务是如何从开始执行到全部执行完毕结束。首先，看到第 55行 read_intr()函数中的 `port_read()`函数，该是一段C语言内嵌汇编指令。其将 256个内存字即一个扇区中 512字节数据读入到请求项缓存结构 buffer中（即缓存中的一个块）。当一个任务较大不只有一个数据块时，通过第 56~58行设置标志位，移动缓存块指针和递增扇区号以及重置 `do_hd`函数指针等操作在磁盘执行完 port_read()后磁盘控制器再次发出中断时，再次执行中断处理程序开始下一个扇区数据的读取直到读完所有所需的扇区为止。write_intr()函数写入数据与上述过程类似，这里不再累述。当任务全部读写完毕，最后通过执行第63行的 `end_request(1)`函数唤醒等待当前任务的进程，并将当前任务指针指向请求队列中的下一个任务。然后再调用 `do_hd_request()`开始下一个任务的执行，依此循环往复直到队列中的所有请求项执行完毕，队首指针为 NULL。最后，当调用 `do_hd_request()`执行到第9行 `INIT_REQUEST`发现没有可执行请求主动退出循环，即完成该设备请求队列中所有任务。上述过程的可参照一下图解：

<div align="center"><img src="/img/in-post/content/os/file-system/reading-process.png" width="90%"/><b>Figure 3：Reading process</b></div>

<div align="center"><img src="/img/in-post/content/os/file-system/writing-process.png" width="90%"/><b>Figure 4：Writing process</b></div>

&emsp;&emsp;以上便是磁盘块设备请求队列中任务循环执行的全过程，在其中一个设备的循环执行过程起于上述 `add_request()`函数第13行 `(dev->request_fn)()`函数调用，终于当该设备请求队列为空后执行上述 `do_hd_request()`函数第9行的  `INIT_REQUEST`宏函数而终止，上述整个过程如下图所示:

<div align="center"><img src="/img/in-post/content/os/file-system/driver-workflow.png" width="90%"/><b>Figure 5：Driver workflow</b></div>

### 3 小结

---
- 本文首先介绍了计算机文件系统的相关概念。然后，以自底向上的顺序开始讲述文件系统的第一部分也是本文的主要内容——设备驱动程序，作为接口程序它为操作系统与具体外设交互提供了具体控制实现。本文作为文件系统系列文章中的第一篇主要是从数据输入输出源头开始介绍文件系统，后续文章将对文件系统中的缓存以及文件组织结构等更多文件系统的其他模块进行讲述，有兴趣的读者可通过后文的阅读进行相关内容的学习。

  ​




### 参考资料
---
- [Linux内核完全注释，赵炯](https://book.douban.com/subject/1231236/)
- [哈尔滨工业大学操作系统课程，李治军](https://www.bilibili.com/video/av17036347?from=search&seid=11186295937821986776)


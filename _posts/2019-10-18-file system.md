---
layout:     post
title:      "操作系统（六）：文件系统（三）"
subtitle:   "Minix文件系统之文件视图"
date:       2019-10-18
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 操作系统
      文件系统
---





### 前言
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2019/10/18/file-system/)
>
> 本文作为 Minix文件系统系列文章的最后一篇，将在前两篇的基础上对文件系统的统一文件视图结构进行介绍。这也是文件系统中最为核心和强大之处，它通过提供了一个统一的文件抽象结构来扩展计算机对所有外设的支持，让计算机变得前所未有的强大！                                                                                                                          



### 1 文件系统结构？
---
&emsp;&emsp;Minix文件系统与标准的 UNIX的文件系统基本相同，它由 6部分组成，在一块硬盘上其分布结构如下图：

<div align="center"><img src="/img/in-post/content/os/file-system/minix-structure.png" width="80%"/><b>Figure 1：Minix File System</b></div>

#### 1.1 超级块

&emsp;&emsp;接下来我们通过 Minix文件系统中几个最核心的数据结构来对其进行深入认识。其中第一个要说的就是**超级块**。其数据结构如下：

```c
/*linux-0.11/include/linux/fs.h */
struct super_block {
	unsigned short s_ninodes;         /* total number of i-nodes */
	unsigned short s_nzones;          /* total number of blocks*/
	unsigned short s_imap_blocks      /* number of blocks occupied by i-node bitmap*/
	unsigned short s_zmap_blocks;     /* number of blocks occupied by the logical block bitmap*/
	unsigned short s_firstdatazone;   /* first block number in data zone*/
	unsigned short s_log_zone_size;   
	unsigned long s_max_size;         /* max file length*/
	unsigned short s_magic;           
/* These are only in memory */
	struct buffer_head * s_imap[8];   /* i-node bitmap pointer in buffer*/
	struct buffer_head * s_zmap[8];   /* block bitmap pointer in buffer*/
	unsigned short s_dev;             /* super block's dev number*/
	struct m_inode * s_isup;
	struct m_inode * s_imount;
	unsigned long s_time;
	struct task_struct * s_wait;
	unsigned char s_lock;
	unsigned char s_rd_only;
	unsigned char s_dirt;
};
```

&emsp;&emsp;以上就是 Minix文件系统超级块数据结构，一种文件系统对应一个超级块，其主要用来记录该文件系统的一些重要属性，例如最大文件长度，文件系统逻辑块数目等等。一个操作系统其文件系统很多时候并不单一，例如 Linux-0.11就包括FAT32，NTFS，Minix以及 EXT2这四种文件系统。在操作系统中每种文件系统都有自身的作用，虽然本文仅介绍 Minix文件系统的一些特性，但没有关系！很多事物的具体实现也许不同但思想有相通之处，学习 Minix不也会为了对基本的文件系统有一个较为直观的认识吗？回到正题，每种文件系统都应该有一个超级块结构，因此 Linux-0.11在内存中通过一个 super_block数据的形式来存储各文件系统的超级块信息。

&emsp;&emsp;超级块结构大部分字段上文都有注释所以不再赘述，接下来仅对其中几个重要属性再强调一下。**i-节点位图**用于说明 i 节点是否被使用，对于 1K大小的盘块来说，可表示 8192个 i-节点使用状况。s_imap[8]表示 **i-节点位图**在高速缓存块中的指针数组。**逻辑块位图**用于描述盘上每个数据块的使用情况，当一个数据盘块被占用时，逻辑块位图中相应比特位被置位。s_zmap[8]表示**逻辑块位图**在高速缓存块中的指针数组，每块缓存块大小是 1024字节，每比特表示一个盘块的占用状态，因此 Minix1.0文件系统所支持的最大块设备容量为 64MB。

&emsp;&emsp;另外需要注意的是，上述结构中第 2~9行的字段在硬盘和内存都同时存在，而第11~20行的字段仅存储在内存中。究其原因主要是第11~20行字段是文件系统加载入内存中后动态使用过程中的一些状态信息，这些信息并非文件系统的静态属性存储在硬盘中没有价值。

#### 1.2 i-节点

&emsp;&emsp;除了超级块（super block）外，Minix文件系统中的 **i-节点**也是非常重要的一个概念。其结构如下所示：

```c
struct m_inode {
	unsigned short i_mode;       /* file mode*/
	unsigned short i_uid;        /* user's id of the file*/
	unsigned long i_size;        /* file length*/
	unsigned long i_mtime;       /* modified time*/
	unsigned char i_gid;         /* owner's group id*/
	unsigned char i_nlinks;      /* link number*/
	unsigned short i_zone[9];    /* array of blocks*/
/* these are in memory also */
	struct task_struct * i_wait; /* the process waited for it*/
	unsigned long i_atime;       /* last visited time*/
	unsigned long i_ctime;       /* modified time of i-node*/
	unsigned short i_dev;        /* device number where the i-node is located*/
	unsigned short i_num;        /* i-node id*/
	unsigned short i_count;      /* reference time*/
	unsigned char i_lock;        /* mark whether it's locked*/
	unsigned char i_dirt;        /* mark whether it's dirty*/
	unsigned char i_pipe;        /* mark whether it's a pipe file */
	unsigned char i_mount;       /* mark whether it mount other filesystem*/
	unsigned char i_seek;        /* search mark*/
	unsigned char i_update;      /* mark whether it's updated*/
};

struct file {
	unsigned short f_mode;       /* file mode*/
	unsigned short f_flags;      /* file open and control mark*/
	unsigned short f_count;      /* reference times*/
	struct m_inode * f_inode;    /* i-node pointer*/
	off_t f_pos;                 /* position of file pointer*/
};

extern struct file file_table[NR_FILE];
```

&emsp;&emsp;每个 i-节点与一个文件一一对应，如上`struct file`文件结构所示，文件中的数据放在磁盘块的数据区中，文件通过其对应的 i-节点与这些数据磁盘块相关联。关联方式就是通过 i-节点中逻辑块数组 i_zone[]存放所关联盘块号实现。其中，i_zone[0]到 i_zone[6]用于存放文件开始的 7个磁盘块号，称为**直接块**。若文件长度小于 7K字节，则可快速查找到对应盘块。若文件大一些时，就需要用到一次间接（i_zone[7]），该盘块中存放着附加的盘块号。对于 Minix文件系统它可以存放 512个盘块号（盘块大小是 1K，而一个块号占 2字节），因此可寻址 512个盘块。若文件还要大，则需要使用二次间接盘块（i_zone[8]）。二次间接块中的一级盘块作用类似于一次间接盘块，所以使用二次间接盘块可以寻址 512x512个盘块。上述过程可参考下图：

<div align="center"><img src="/img/in-post/content/os/file-system/file-view.png" width="90%"/><b>Figure 2：File view</b></div>


### 2 文件操作
---
&emsp;&emsp;在上节对文件相关数据结构进行分析的基础上，本节将通过文件相关系统调用来认识整个统一的文件视图。

#### 2.1 读写文件系统调用

&emsp;&emsp;由之前[操作系统(三):系统调用](https://steverao.github.io/2019/10/08/syscall-flow/)和[操作系统实验(三):系统调用](https://steverao.github.io/2019/11/07/syscall-lab/)相关内容的基础可知。平时我们在C或其他高级语言中使用的类似于 printf(), scanf()以及 fopen()等文件操作函数底层是通过调用操作系统提供的 write()和 read()等系统调用来实现的。接下来我们通过 Linux-0.11中 write()和 read()系统调用的实现 sys_write()和sys_read()来看看操作系统到底是如何实现统一的文件视图：

```c
/*linux-0.11/fs/read_write.c */
int sys_read(unsigned int fd,char * buf,int count)
{
	struct file * file;
	struct m_inode * inode;

	if (fd>=NR_OPEN || count<0 || !(file=current->filp[fd]))
		return -EINVAL;
	if (!count)
		return 0;
	verify_area(buf,count);
	inode = file->f_inode;
	if (inode->i_pipe)
		return (file->f_mode&1)?read_pipe(inode,buf,count):-EIO;
	if (S_ISCHR(inode->i_mode))
		return rw_char(READ,inode->i_zone[0],buf,count,&file->f_pos);
	if (S_ISBLK(inode->i_mode))
		return block_read(inode->i_zone[0],&file->f_pos,buf,count);
	if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode)) {
		if (count+file->f_pos > inode->i_size)
			count = inode->i_size - file->f_pos;
		if (count<=0)
			return 0;
		return file_read(inode,file,buf,count);
	}
	printk("(Read)inode->i_mode=%06o\n\r",inode->i_mode);
	return -EINVAL;
}
/*linux-0.11/fs/read_write.c */
int sys_write(unsigned int fd,char * buf,int count)
{
	struct file * file;
	struct m_inode * inode;
	
	if (fd>=NR_OPEN || count <0 || !(file=current->filp[fd]))
		return -EINVAL;
	if (!count)
		return 0;
	inode=file->f_inode;
	if (inode->i_pipe)
		return (file->f_mode&2)?write_pipe(inode,buf,count):-EIO;
	if (S_ISCHR(inode->i_mode))
		return rw_char(WRITE,inode->i_zone[0],buf,count,&file->f_pos);
	if (S_ISBLK(inode->i_mode))
		return block_write(inode->i_zone[0],&file->f_pos,buf,count);
	if (S_ISREG(inode->i_mode))
		return file_write(inode,file,buf,count);
	printk("(Write)inode->i_mode=%06o\n\r",inode->i_mode);
	return -EINVAL;
}
```

&emsp;&emsp;以上代码就是通过统一的文件结构来管理计算机外设的核心代码。因为读/写逻辑类似，所以我们仅通过对 sys_read()讲解来认识整个过程。首先，参数中 fd是将读取的文件的文件句柄（在内核中，文件句柄即文件的唯一标识，操作系统中有 sys_open()这一系统调用可提供文件名到文件句柄 fd之间的转换，在操作文件前一般先要打开该文件，这也与我们在C语言中操作文件认知一致），每个进程中都有一个对应该进程的文件结构数组，用于记录该进程已打开的文件结构体，如上述代码第 7行 `filp[]`所示。有了句柄就可通过其找到对应的文件结构体和 i-节点等文件描述结构开始对文件进行操作。第二个参数字符指针 buf它表示用户缓冲区。当要进行写文件时，待写入的数据就存放在该指针所指向的缓冲区内，读文件时，最终从文件外设中读出的数据也存放在其中。最后一个字符 count表示将读写字符的数目。

&emsp;&emsp;第7行通过文件句柄 fd获取对应文件的描述符，其结构上文已介绍。第12行通过 file结构体获得其对应的 i-node节点。接下来就是该函数的核心逻辑，通过 `inode->i_mode`属性判断文件所述设备类型。当是字符设备文件时，就调用  `rw_char()`字符文件操作函数，对底层的控制台，显示器，鼠标以及键盘等字符设备进行读写控制。当为块设备文件时，通过 `block_read()`块设备操作函数调用块缓存来读写数据块。另外其他如管道等设备类型也与上述两种最典型设备的处理过程有类似，这里就不赘述，有兴趣的读者可以参考相关源代码。上述过程可表示为下图：

<div align="center"><img src="/img/in-post/content/os/file-system/sys_read.png" width="80%"/><b>Figure 3：System call</b></div>



#### 2.2 块设备

&emsp;&emsp;在文件系统上一篇文章[数据块缓存](https://steverao.github.io/2019/10/14/file-system/)最后小结部分提出了文件系统如何通过统一的文件抽象获取需读写块的 dev和 block来调用缓存接口 geblk()实现管理块设备这一问题？接下来通过上文提到的 block_write()函数来回答上述问题（block_read()函数也类似就不在赘述）。

```c
int block_write(int dev, long * pos, char * buf, int count)
{
	int block = *pos >> BLOCK_SIZE_BITS;
	int offset = *pos & (BLOCK_SIZE-1);
	int chars;
	int written = 0;
	struct buffer_head * bh;
	register char * p;

	while (count>0) {
		chars = BLOCK_SIZE - offset;// size can be written in the block
		if (chars > count)
			chars=count;
		if (chars == BLOCK_SIZE)
			bh = getblk(dev,block);
		else
			bh = breada(dev,block,block+1,block+2,-1);
		block++;
		if (!bh)
			return written?written:-EIO;
		p = offset + bh->b_data;
		offset = 0;
		*pos += chars;
		written += chars;
		count -= chars;
		while (chars-->0)
			*(p++) = get_fs_byte(buf++);//write the data into cache block from user cache.
		bh->b_dirt = 1;
		brelse(bh);
	}
	return written;
}
```

&emsp;&emsp;block_write()函数参数中 pos指针对应 file结构中的 f_pos属性，表示将读写块在设备文章中的偏移量的指针。第 3行通过该指针的值右移 BLOCK_SIZE_BITS=10位来获得将写入的数据块的 block号（右移 10位相当于除以 1024，即一个块的大小）。第四行根据 pos指针计算将写入数据块的偏移值（BLOCK_SIZE=1024）。第 10~30行就是通过 dev和 block调用 getblk()函数获取目标缓存数据块，然后通过第 26~27行调用 `get_fs_byte()`函数将用户态缓存空间中待写入数据写到内核态数据缓存上（breada()函数底层也是调用 getblk()来获取缓存块）。最后，通过 brelse()函数将缓存中数据写到硬盘等块设备上。这样就与上篇文章的缓存接口联系上了，打通了从文件系统调用到操纵对应块设备的整个流程。



### 3 小结
---
- 本文首先介绍了文件系统结构**文件**在 Linux-0.11操作系统中的组织形式，然后在前两篇关于文件系统文章基础上通过文件读写系统调用 sys_read()/sys_write()来展示操作系统如何通过统一的文件视图来管理各类计算机外设。

- 这三篇文件系统系列文章主要还是围绕第一篇文章开头的那张统一的文件系统结构图来展开的，由于篇幅和时间等因素制约，该系统文章核心主要还是围绕块设备相关内容展开。对于其他的例如字节设备驱动相关的程序本系列文章没有进行太多分析，不过其实现相较于块设备而言它们更加简单，有兴趣的读者可以参考相关资料进行学习。

  ​




### 参考资料
---
- [Linux内核完全注释，赵炯](https://book.douban.com/subject/1231236/)
- [哈尔滨工业大学操作系统课程，李治军](https://www.bilibili.com/video/av17036347?from=search&seid=11186295937821986776)


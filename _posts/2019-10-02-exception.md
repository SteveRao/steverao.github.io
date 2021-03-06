---
layout:     post
title:      "操作系统（一）：异常"
subtitle:   "中断，陷阱，故障"
date:       2019-10-02
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 操作系统
---





### 总体概述
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2019/10/02/exception/)
>
> 操作系统层面的异常被认为是操作系统设计中里程碑式的成果，其是操作系统实现I/O、进程和虚拟内存机制以及用户态与内核态切换的基本机制。对其良好的认识对于理解其他计算机领域的知识与技术有着重要意义。
>
> **本文遵循[CC-BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)开源分享协议，转载文章内容请注明出处。**                                                                                                                       



### 1 操作系统“异常”是什么？
---
&emsp;&emsp;**“异常”**这个词相信对很多学习过像C++，Java这类高级程序设计语言的人来说应该不陌生。但本文所要讲述的**异常**并非就是高级程序设计语言中的异常，但他们确实也有联系，后文对其间的关系进行介绍。对于操作系统中的异常，《深入理解计算机系统（第三版）》一书给出的定义如下：

```Html
异常(Exception)就是控制流（处理器执行序列）中的突变，用来响应处理器状态中的某些变化。
```

&emsp;&emsp;结合下图便可较为清晰的理解上述概念：

<div align="center"><img src="/img/in-post/content/exception/exception.png" width="70%"/><b>异常</b></div>

&emsp;&emsp;由图可知，异常就是响应处理器被某个**事件**打断正常程序执行序列的过程。这些事件可以和当前执行的指令相关，比如，发生虚拟内存缺页，算术溢出，或者一条指令试图除以零。另外这些事件也可与当前的指令执行无关，比如，一个系统定时器产生信号或者一个I/O请求完成。



### 2 异常类别
---
&emsp;&emsp;操作系统**异常**根据其信号的发起源可以分成**中断(Interupt)**，**故障(Fault)**和**陷阱(Trap)**。*（后文中所涉处理器内容，其对应的处理器默认为Intel x86系列）*

#### 2.1 中断

&emsp;&emsp;**中断**是由外部硬件向处理器发起的异常信号，所以也叫外部硬件中断。在系统正常运行的过程中，当外部设备发生错误，或者有数据要传送（比如，从网络中接收到一个针对当前主机的数据包），或者处理器交给它的事情处理完了（比如，打印已经完成），它们都会拍一下处理器的肩膀，告诉它应当先把手头上的事情放一放，来临时处理一下。其具体执行示意图如下：

<div align="center"><img src="/img/in-post/content/exception/interupt.png" width="70%"/><b>中断</b></div>

#### 2.2 故障

&emsp;&emsp;与由外部硬件所引发的中断不同，**故障**发生在处理器内部，是由执行的指令引起的。常见如内存缺页，算术溢出和除零等异常都属于故障。这也就是C++，Java等高级程序语言中常说的异常概念。其具体执行示意图如下：

<div align="center"><img src="/img/in-post/content/exception/fault.png" width="70%"/><b>故障</b></div>

#### 2.3 陷阱

&emsp;&emsp;与上述两种异常由意外事件引发不同的是，**陷阱**是**有意的**异常，是由执行一条执行在处理器内部的指令引发的结果。陷阱最重要的用途就是在用户程序和系统内核之间提供一个像过程调用一样的接口，叫**系统调用**。系统调用是操作系统为用户程序利用系统常见资源所提供的接口，其涉及到操作系统中非常多重要概念，后续也会单独通过一篇博文来详细对其介绍。Linux环境中常见的系统调用如，读写文件的*read/write*，创建执行进程的*fork/execve*等。其具体执行示意图如下：

<div align="center"><img src="/img/in-post/content/exception/trap.png" width="70%"/><b>陷阱</b></div>

#### 2.4 小结
&emsp;&emsp;根据对于以上三类异常的描述，对其相关概念小节如下表：

|  类别  |     原因      | 异步/同步 |   返回行为    |
| :--: | :---------: | :---: | :-------: |
|  中断  |  来自硬件设备的信号  |  异步   | 总返回到下一条指令 |
|  故障  | 潜在可/不可恢复的错误 |  同步   | 返回当前指令或终止 |
|  陷阱  |    有意的异常    |  同步   | 总返回到下一跳指令 |

&emsp;&emsp;其中，异步异常是指由处理器外部的I/O设备中的事件产生的，同步异常是执行一条指令的直接产物。




### 3 异常处理
---
&emsp;&emsp;对于系统中的每一种可能的异常，操作系统为其都分配了一个唯一的**异常号**。其中一部分号码由处理器的设计者分配，其他号码由操作系统内核的设计者分配。前者示例包括除零，缺页，内存访问违章和断点等。后者示例包括系统调用和外部I/O设备信号等。

&emsp;&emsp;在操作系统启动过程中，系统会初始化一张异常表，其内容就是系统异常号与对应异常处理程序所在地址的一张映射表。表结构如下图：

<div align="center"><img src="/img/in-post/content/exception/exception-table.png" width="70%"/><b>异常表</b></div>

&emsp;&emsp;在Intel x86系统中，异常表项数目为256，在满足基本使用需求的情况下，还有一些预留项。结合上述异常图和异常表图可知异常处理的过程如下：

1. 根据处理器被打断的具体事件，系统产生一个异常，并将相应的异常号传递给处理器。

2. 处理器根据所获得的异常号到内存中异常表中检索获得对应异常处理程序的执行地址。

3. 处理器根据异常处理程序的地址跳转去执行相应的异常处理，并根据异常的形式返回或不返回结果（对应系统终止）。

   ​



### 4 异常应用案例分析
---
&emsp;&emsp;为了更深入的理解异常相关概念，本部分结合具体的**读键盘字符并显示**的汇编代码来认识与我们日常使用计算机最息息相关的异常——陷阱。

```java
;文件说明：采用陷阱来实现从键盘读数据并输出到显示器 
;===============================================================================
SECTION header vstart=0                     ;定义用户程序头部段 
    program_length  dd program_end          ;程序总长度[0x00]
    
    ;用户程序入口点
    code_entry      dw start                ;偏移地址[0x04]
                    dd section.code.start   ;段地址[0x06] 
    ;部分变量定义省略
    
header_end:                
;===============================================================================
SECTION code align=16 vstart=0           ;定义代码段（16字节对齐） 
start:
      mov ax,[stack_segment]
      mov ss,ax
      mov sp,ss_pointer
      mov ax,[data_segment]
      mov ds,ax
      mov cx,msg_end
      mov bx,message
      
 .putc:
      mov ah,0x0e    
      mov al,[bx]
      int 0x10       
      inc bx
      loop .putc

 .reps:
      mov ah,0x00
      int 0x16       
      
      mov ah,0x0e
      mov bl,0x07
      int 0x10       
      jmp .reps
;===============================================================================
SECTION data align=16 vstart=0

    message       db 'Hello, friend!'
                  db 'This simple procedure used to demonstrate '
                  db 'the BIOS interrupt.',0x0d,0x0a
                  db 'Please press the keys on the keyboard ->'
                 
    msg_end:                 
;===============================================================================
SECTION stack align=16 vstart=0
           
                 resb 256
ss_pointer:
;===============================================================================
SECTION program_trail
program_end:
```

&emsp;&emsp;这段代码只是为了讲述案例，为了方便讲述其中省略了一些对于讲解案例没用的变量定义等冗余内容。

&emsp;&emsp;首先，程序的主干逻辑是14~37行，最上面的header段和最下面的data段都可以先不关注。15~21行是初始化代码执行所需的寄存器，需要关注的是21行将data段41行将显示在屏幕中的字符串的首地址放到了bx寄存器中。24行将ax寄存器高8位ah置为0x0e，再将第一个8位的字符存到ax寄存器低8位al中，然后，ax寄存器内容作为参数再结合陷阱**int 0x10**引发异常，让处理器在屏幕输出一个字符。27~28通过loop循环将message字符串内容在屏幕中全部输出。

&emsp;&emsp;接着，31~32行采用陷阱**int 0x16**从键盘中读入一个字符。34~36行采用陷阱**int 0x10**将刚读入的字符显示在屏幕中。30~37行核心作用就是循环将键盘打下的字符显示在屏幕中。

&emsp;&emsp;程序结果显示如下图：

<div align="center"><img src="/img/in-post/content/exception/result.png" width="70%"/><b>运行结果</b></div>

&emsp;&emsp;由上述案例分析可知，应用程序想要使用计算机I/O硬件资源需要通过陷阱（系统调用）来实现。其与我们日常如此息息相关，以至于我们在使用计算机过程中的时时刻刻都在通过异常让处理器来响应我们发起的任务。

### 5 小结
---
- 本文从操作系统异常定义，到异常类型，再到异常处理过程和异常示例分析的顺序来介绍了操作系统中的**异常**概念。本文是对笔者这段时间学习操作系统底层原理中异常部分的一个总结。

- 在讲操作系统设计的时候，说进程和线程是操作系统中的伟大设计相信很多人应该没有太多异议。但是否考虑过进程和线程中的控制流切换思想其实与本文的异常是一样。所以这里抛出一个问题，异常所引发的处理器切换与进程或线程引发的处理器切换之间有什么异同？

  ​



### 参考资料
---
- [《x86汇编语言：从实模式到保护模式》,李忠](https://book.douban.com/subject/20492528//)
- [《深入理解计算机系统（第三版）》,Randal E.Bryant](https://book.douban.com/subject/26912767/)
- [哈尔滨工业大学操作系统课程，李治军](https://www.bilibili.com/video/av17036347?from=search&seid=11186295937821986776)
- [操作系统原理与实践，李治军](https://www.shiyanlou.com/courses/115)
- [Linux下的中断机制](https://lrita.github.io/2019/03/05/linux-interrupt-and-trap/)


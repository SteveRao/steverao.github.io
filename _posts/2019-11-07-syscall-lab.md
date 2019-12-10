---
layout:     post
title:      "操作系统实验（三）：系统调用"
subtitle:   "系统调用实验"
date:       2019-11-07
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 操作系统实验
---





### 前言
---
> ***I hear and I forget, I see and I remember, I do and I understand！***
>
> 本文是哈工大李志军老师操作系统实验系列实验二：“系统调用”实验过程的记录*（因为仅是实验记录，文中实验原理性解释较少）*                                                                                         



### 1 实验目的
---
- 建立对系统调用接口的深入认识；

- 掌握系统调用的基本过程；

- 能完成系统调用的全面控制；

- 为后续实验做准备；

  ​

### 2 实验内容
---
1. 在Linux 0.11内核中添加一个名叫`iam()`的系统调用:

   ```c
   int iam(const char * name)
   ```

   完成的功能是将字符串参数 name 的内容拷贝到内核空间中保存下来。要求 name 的长度不能超过 23 个字符。返回值是拷贝的字符数。如果 name的字符个数超过了 23，则返回 “-1”，并置 errno 为 EINVAL。

2. 在Linux 0.11内核中添加一个名叫`whoami()`的系统调用:

   ```c
   int whoami(char* name, unsigned int size)
   ```

   完成的功能是将内核中由 iam() 保存的名字拷贝到 name 指向的用户地址空间中，同时确保不会对 name越界访存（name 的大小由 size 说明）。返回值是拷贝的字符数。如果 size小于需要的空间，则返回“-1”，并置 errno 为 EINVAL。

3. 编写测试程序，完成对上述两个系统调用功能的测试。

   ​

### 3 实验步骤
---
&emsp;&emsp;本次实验的环境请参考[操作系统实验（一）：环境搭建](https://steverao.github.io/2019/11/03/oslab-environment/)，由于实验需要修改的是Linux 0.11系统内核，所以实验需要改动的文件都在linux-0.11目录下。本次实验一共需要改动**unistd.h**，**sys.h**，**system_call.s**和**Makefile**内核四个文件，并新增**who.c**，**iam.c**和**whoami.c**三个文件。详细实验步骤如下：

- 系统调用通过陷阱机制（软中断）从用户态进入内核态进而在内核中调用对应的异常处理程序，所以先为新增的系统调用设置**调用号**。修改linux-0.11/include/unistd.h

  ```c
  #define __NR_setup	0	/* used only by init, to get system going */
  #define __NR_exit	1
  #define __NR_fork	2
  #define __NR_read	3
  #define __NR_write	4
  ......
  #define __NR_sgetmask	68
  #define __NR_ssetmask	69
  #define __NR_setreuid	70
  #define __NR_setregid	71

  /* 新增系统调用调用号*/
  #define __NR_iam	72     
  #define __NR_whoami	73
  ```


- 将系统调用总数从72修改成74，修改linux-0.11/kernel/system_call.s

  ```c
  # offsets within sigaction
  sa_handler = 0
  sa_mask = 4
  sa_flags = 8
  sa_restorer = 12

  /* 修改nr_system_calls */
  nr_system_calls = 74
  ```


- 为新增的系统调用添加**系统调用名**并维护**系统调用表**，修改linux-0.11/include/linux/sys.h

  ```c
  extern int sys_setup();
  extern int sys_exit();
  extern int sys_fork();
  extern int sys_read();
  ......
  extern int sys_sgetmask();
  extern int sys_ssetmask();
  extern int sys_setreuid();
  extern int sys_setregid();

  /* 新增系统调用名 */
  extern int sys_iam();
  extern int sys_whoami();

  /* 为新增系统调用维护系统调用表 */
  fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
  sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
  sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
  sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
  sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
  sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
  sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
  sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
  sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
  sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
  sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
  sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
  sys_setreuid,sys_setregid, sys_iam, sys_whoami };
  ```


- 为新增的系统调用编写代码实现，在linux-0.11/kernel目录下，创建一个文件who.c，该实现过程会用到两个内核已经提供的函数**get_fs_byte()**和**put_fs_byte()**，具体实现如下

  ```c
  #define __LIBRARY__  
  #include <unistd.h>  
  #include <errno.h>  
  #include <asm/segment.h>
  #include <string.h> 
        
  char username[64]={0};  
        
  int sys_iam(const char* name)  
  {  
      int i = 0;  
      /* get_fs_byte() 获得一个字节的用户空间中的数据*/
      while(get_fs_byte(&name[i]) != '\0')
        i++;  
      if(i > 23)
        return -EINVAL;  
      i=0;  
      while(1)
      {
         username[i] = get_fs_byte(&name[i]);
         if(username[i] == '\0')
           break;  
         i++;
      }    
      return i;  
  }   
        
  int sys_whoami(char* name,unsigned int size)  
  {  
       int i,len;  
       len = strlen(username);  
       if (size < len)  
         return -1;  
       i=0;  
       while(i < len)
       {  
           /*put_fs_byte（）从内核空间中拷贝一个字节到用户空间*/
           put_fs_byte(username[i],&name[i]);  
           i++;  
       }      
       return i;  
  } 
  ```


- 将新增的系统调用实现who.c编译到linux-0.11内核中，修改linux-0.11/kernel/Makefile文件如下两处：

  其一：

  ```makefile
  OBJS  = sched.o system_call.o traps.o asm.o fork.o \
          panic.o printk.o vsprintf.o sys.o exit.o \
          signal.o mktime.o
  ```

  改成：

  ```makefile
  OBJS  = sched.o system_call.o traps.o asm.o fork.o \
          panic.o printk.o vsprintf.o sys.o exit.o \
          signal.o mktime.o who.o
  ```

  其二：

  ```makefile
  ### Dependencies:
  exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
    ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
    ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
    ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
    ../include/asm/segment.h
  ```

  改成：

  ```makefile
  ### Dependencies:
  who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
  exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
    ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
    ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
    ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
    ../include/asm/segment.h
  ```

  修改完成后，在linux-0.11/kernel/目录下执行以下命令将who.c编译到内核中

  ```Shell
  make
  ```

  执行成功之后，目录下会生成一个who.o文件。

- 到此为止，内核中需要修改的部分已经完成，接下来需要编写测试程序来验证新增的系统调用是否已经被编译到linux-0.11内核可供调用。首先在oslab目录下编写iam.c，whoami.c和testlab2.sh三个文件，文件内容分别如下：

  ```c
  /* iam.c */
  #define __LIBRARY__
  #include <unistd.h> 
  #include <errno.h>
  #include <asm/segment.h> 
  #include <linux/kernel.h>
  _syscall1(int, iam, const char*, name);
   
  int main(int argc, char *argv[])
  {
      /*调用系统调用iam()*/
      iam(argv[1]);
      return 0;
  }
  ```

  ```c
  /* whoami.c */
  #define __LIBRARY__
  #include <unistd.h> 
  #include <errno.h>
  #include <asm/segment.h> 
  #include <linux/kernel.h>
  #include <stdio.h>
   
  _syscall2(int, whoami,char *,name,unsigned int,size);
   
  int main(int argc, char *argv[])
  {
      char username[64] = {0};
      /*调用系统调用whoami()*/
      whoami(username, 24);
      printf("%s\n", username);
      return 0;
  }
  ```

  ```shell

  /* testlab2.sh */
             
  gcc iam.c -o iam 
  gcc whoami.c -o whoami
  ./iam raozihao
  ./whoami   
  ```

- 以上三个文件需要放到启动后的linux-0.11操作系统上运行，验证新增的系统调用是否有效，那如何才能将这三个文件从宿主机转到稍后虚拟机中启动的linux-0.11操作系统上呢？这里我们采用挂载方式实现宿主机与虚拟机操作系统的文件共享，在oslab/目录下执行以下命令挂载hdc目录到虚拟机操作系统上。

  ```Shell
  sudo ./mount-hdc 
  ```

  再通过以下命令将上述三个文件拷贝到虚拟机linux-0.11操作系统/usr/root/目录下，命令在oslab/目录下执行：

  ```Shell
  cp iam.c whoami.c testlab2.sh hdc/usr/root
  ```

  如果目标目录下存在对应的三个文件则可启动虚拟机进行测试了。

- 在oslab下执行./run启动虚拟机后，在/usr/root/目录下执行以下命令进行测试

  ```Shell
  testlab2.sh
  ```

  命令执行后，很可能会报以下错误：

  <div align="center"><img src="/img/in-post/content/oslab/syscall/fault.png" width="70%"/><b>异常</b></div>

  这代表虚拟机操作系统中/usr/include/unistd.h文件中没有新增的系统调用调用号，使用如下命令编辑文件：

  ```Shell
  vi /usr/include/unistd.h
  ```

  为新增系统调用设置调用号*（该处内容与步骤一保持一致）*

  ```c
  /* 新增系统调用调用号*/
  #define __NR_iam	72     
  #define __NR_whoami	73
  ```

  最后，再执行testlab2.sh脚本进行测试，结果如下则表示成功*（这里提醒一下，在虚拟机linux-0.11操作系统上操作一定要小心和迅速，不然系统可能卡顿导致无法修改或正常显示结果）：*

  <div align="center"><img src="/img/in-post/content/oslab/syscall/success.png" width="70%"/><b>调用成功</b></div>

  ​

### 参考资料
---
- [哈尔滨工业大学操作系统课程，李治军](https://www.bilibili.com/video/av17036347?from=search&seid=11186295937821986776)
- [操作系统实践](https://www.kancloud.cn/digest/os-experiment/120078)
- [哈工大操作系统实验三：系统调用](https://blog.csdn.net/yuebowhu/article/details/78755728)
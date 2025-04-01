---
title: "Debugging and troubleshooting methods for userspace-rcu."
date: 2020-03-28
---

# userspace-rcu的调试定位方法

rcu最开始是从Linux kernel里面实现的，因为实现了reader无锁编程，能够大幅提高性能。
urcu（userspace-rcu），顾名思义就是用户态的开源的rcu实现库。通过urcu可以实现reader无锁，大大提高应用程序性能。
urcu库使用相对比较简单，但是使用不当也会造成各种crash问题。我们项目中使用了urcu开源库，在调试中遇到并解决了一些问题，下面在使用中的一些总结。希望能有参考意义。

## 1. 开启urcu的debug功能

刚开始调试时，我们希望问题尽早暴露出来并修复。建议使用开启debug功能的urcu lib库，待调试OK后，再使用关闭debug功能的urcu lib库。

### 1.1 编译带debug版本库

.configure命令添加 --enable-rcu-debug 项以开启debug功能。
```powershell
[root@localhost]# ./bootstrap 

[root@localhost]# ./configure --disable-shared --enable-rcu-debug
...//省略部分
Uspace-RCU 0.11.0

Features
  Target architecture:                x86_64
  SMP support:                        yes
  Memory fence instructions:          yes
  Futex support:                      yes
  Thread Local Storage (TLS):         __thread
  clock_gettime():                    yes
  Require membarrier:                 no
  Internal debugging:                 yes
  Lock-free hash table iterator debugging: no
  Multi-flavor support:               yes

Install directories
  Binaries:                           /usr/local/bin
  Libraries:                          /usr/local/lib

[root@localhost]# make && make install

```
### 1.2 使用带debug功能的urcu库

使用liburcu.a编译你的应用程序，然后运行调试。

增加了debug功能，任何线程，只要没有reg urcu thread，在调用 rcureadlock时，会触发assert 。crash栈明显可以看到原因是线程未注册urcu thread。这样可以更早的发现问题线程，避免直接使用正式urcu库，在运行中出现各种诡异的错误问题。

## 2. 使用不同特性的urcu库

urcu库提供了不同特性的urcu实现。

对于常用的include <urcu.h>, 可以有liburcu.a、liburuc-signal.a和liburcu-mb.a三种选择。可以不改动应用程序代码，通过编译选项RCU_MEMBARRIER，RCU_SIGNAL，RCU_MB在这三个中切换。

从下面代码可以看到

```c
// include/urcu.h
#if !defined(RCU_MEMBARRIER) && !defined(RCU_SIGNAL) && !defined(RCU_MB)
#define RCU_MEMBARRIER
#endif

#ifdef RCU_MEMBARRIER
#include <urcu/urcu-memb.h>
#elif defined(RCU_SIGNAL)
#include <urcu/urcu-signal.h>
#elif defined(RCU_MB)
#include <urcu/urcu-mb.h>
#else
#error "Unknown urcu flavor"
#endif
```


### 2.1 如何启用对应特性库，以RCU_MB为例

urcu默认使用RCU_MEMBARRIER。如果我们想要使用RCU_MB，需要在编译选项中增加 -DRCU_MB 来启用。
启用RCU_MB后，连接时需要链接liburcu-mb.a

不同特性的urcu库在性能不尽相同，可以根据实际需要选择合适的库。

## 3. 使用urcu的常见问题

### 3.1 部分线程未注册urcu thread

urcu要求每个会使用urcu功能的线程，在线程开始时 调用 urcu reg thread API， 在结束时调用 urcu un thread API。
实际编码中经常会忘记reg/unreg urcu thread，导致一些不可预期的问题。这个问题可以通过使用debug版本来迅速暴露出来。在debug版本，如果有线程没有注册urcu thread，在使用urcu API时，会被assert捕捉到，将问题迅速暴露出来。

### 3.2 线程开始时注册了urcu thread，但是结束时未注销urcu thread

线程结束时未注销urcu thread，导致线程节点内存已经释放了，但是节点地址还挂在urcu 链表上。在某个时刻就会触发错误的地址引用，导致crash。

基本上大多数问题都出在这两个地方，在编码中需要多加注意。

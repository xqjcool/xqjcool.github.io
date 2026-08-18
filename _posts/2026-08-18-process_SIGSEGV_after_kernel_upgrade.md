---
title: 升级内核版本后，应用进程频繁crash
date: 2026-08-18
---

# 升级内核版本后，应用进程频繁crash

## 一、 问题起因

基于安全团队要求，以及新特性需要，我们定期会升级产品的linux kernel版本。
这次从linux 6.1.62 要直接升级到 linux 6.18 版本。
辛苦更新内核，应用产品相关改动，修复因版本变化带来的各种编译问题，然后再修复运行时问题，最终稳定下来。
交付测试后，发现proxy_daemon在测试过程中会crash，导致不能正常工作。


## 二、问题现象和分析

### 1. 查看 core文件

进程crash存在多个位置，有在libhiredis.so中，有在其他lib中的。相对较为方便分析的是_conn_hash_init这个crash，是我们产品代码。于是着重查看这个core。

gdb打开core初步查看，问题是 hash table初始化过程中出现了SIGSEGV内存异常。

```
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00007f8ebb60da81 in _conn_hash_init (caller=0x55e3b8241bb3 "qc_table_init", bucket_size=bucket_size@entry=131071, compare=<optimized out>, max_nodes=max_nodes@entry=10000, 
--Type <RET> for more, q to quit, c to continue without paging--
    lock_type=lock_type@entry=conn_hash_FO_LOCK_TYPE) at conn_hash.c:381

warning: 381	conn_hash.c: No such file or directory
[Current thread is 1 (LWP 2279)]
(gdb) bt
#0  0x00007f8ebb60da81 in _conn_hash_init (caller=0x55e3b8241bb3 "qc_table_init", bucket_size=bucket_size@entry=131071, compare=<optimized out>, max_nodes=max_nodes@entry=10000, 
    lock_type=lock_type@entry=conn_hash_FO_LOCK_TYPE) at conn_hash.c:381
#1  0x000055e3b8f2ea40 in qc_table_init (addr_table_size=addr_table_size@entry=100000, cid_table_size=cid_table_size@entry=10000) at ./http3_quic/quic_conn.c:621
#2  0x000055e3b85ac1f1 in _worker_init (ti=ti@entry=0x7f8eb1470560) at ./core/worker.c:1398
#3  0x000055e3b85b025c in worker_run (arg=0x7f8eb1470560) at ./core/worker.c:2720
#4  0x00007f8eb41364b1 in start_thread (arg=<optimized out>) at pthread_create.c:454
#5  0x00007f8eb4192248 in __GI___clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78

```

### 2. 分析和尝试-1

经过分析是 hash table 被申请后，刚开始初始化赋值,发现内存就不可访问了。此时怀疑是proxy_daemon进程存在double free或者其他原因导致的UAF，这个 **伏笔** 影响很大，暂时按下不表。
先是尝试review相关代码，没有发现明显的问题。
于是在代码中添加了相关日志，期望能帮助找到线索。debug image 复现后，发现问题比较复杂。有时是内存不可访问，有时是相关内存内容被写坏，导致异常地址访问错误。

为了对比测试，其他所有都不变，只将内核从 linux 6.18 替换回  linux 6.1.62 ，进程运行稳定，没有core。根据这个结果，当时怀疑是新内核的什么机制导致了进程隐藏的bug暴露了。这里也是个 **伏笔** ，按下不表。

### 3. 分析和尝试-2

因为部分core和libjemalloc相关，查看libjemalloc源码，没有发现问题，甚至在libjemalloc开源github上提issue，被告知尝试不开启libjemalloc试试。
尝试没有libjemalloc的版本，一样会出现core。


### 4. 分析和尝试-3

尝试开启asan版本，不知道为什么，没能定位内存问题，反而出现了更多奇怪的问题，导致进程运行不正常。没办法帮助定位问题。

### 5. 分析和尝试-4

因为初期都关注在malloc/alloc/free的异常上。同事建议用gdb attach后使用脚本记录malloc/alloc/free相关地址，出问题后进行日志分析。
因为进程过于庞大，申请释放频繁，导致总是运行中退出。

### 6. 分析和尝试-5

逐渐发现问题都发生在大内存的释放，并且会调用munmap，尝试在内核中加入munmap日志，等core出现时，看看对于内存时不是被unmap了。
经过多次复现，确实有发现core时的内存，在crash前已经被unmap了。
为了找到被谁unmap的，对unmap日志优化，打印用户态部分站内容。然后在分析core时通过gdb查看栈中的函数。
发现hyperscan的库里有个object释放了这个指针，尝试分析hyperscan的库，也没有发现问题原因，在hyperscan的github体issue，没人搭理 :-(



## 三、柳暗花明，转机到来

经过很多次的复现crash，分析，修改尝试。再复现crash，分析，修改尝试。
觉得只打印munmap，无法知道这个内存的由来。我们想要看看它什么时候被map的，什么时候被unmap的完整时间线。
于是增加了mmap的信息日志打印。

再次复现后，查看core

```

```



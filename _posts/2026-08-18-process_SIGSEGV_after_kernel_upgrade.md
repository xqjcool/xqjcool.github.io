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


### 7. 分析和尝试-6

尝试了一些可能的修改，发现还是有core，gdb查看core

```
(gdb) bt
#0  0x00007f8ebb60da81 in _chash_init (caller=0x55e3b8241bb3 "qc_conn_table_init", bucket_size=bucket_size@entry=131071, compare=<optimized out>, max_nodes=max_nodes@entry=10000, 
    lock_type=lock_type@entry=CHASH_FO_LOCK_TYPE) at chash.c:381
#1  0x000055e3b8f2ea40 in qc_conn_table_init (addr_table_size=addr_table_size@entry=100000, cid_table_size=cid_table_size@entry=10000) at ./http3_quic/quic_conn.c:621
#2  0x000055e3b85ac1f1 in _worker_init (ti=ti@entry=0x7f8eb1470560) at ./core/worker.c:1398
#3  0x000055e3b85b025c in worker_run (arg=0x7f8eb1470560) at ./core/worker.c:2720
#4  0x00007f8eb41364b1 in start_thread (arg=<optimized out>) at pthread_create.c:454
#5  0x00007f8eb4192248 in __GI___clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78

(gdb)   p $_siginfo
$1 = {si_signo = 11, si_errno = 0, si_code = 1, _sifields = {_pad = {1305972736, 32654, 0 <repeats 26 times>}, _kill = {si_pid = 1305972736, si_uid = 32654}, _timer = {si_tid = 1305972736, si_overrun = 32654, 
      si_sigval = {sival_int = 0, sival_ptr = 0x0}}, _rt = {si_pid = 1305972736, si_uid = 32654, si_sigval = {sival_int = 0, sival_ptr = 0x0}}, _sigchld = {si_pid = 1305972736, si_uid = 32654, si_status = 0, 
      si_utime = 0, si_stime = 0}, _sigfault = {si_addr = 0x7f8e4dd79000}, _sigpoll = {si_band = 140249168056320, si_fd = 0}, _sigsys = {_call_addr = 0x7f8e4dd79000, _syscall = 0, _arch = 0}}}

(gdb)   x/i $pc
=> 0x7f8ebb60da81 <_chash_init+465>:	mov    %r14,(%r12,%rax,1)
(gdb)   info registers
rax            0xf8000             1015808
rbx            0xf800              63488
rcx            0x55e3fadc3b00      94437654674176
rdx            0xf800              63488
rsi            0x1                 1
rdi            0x1e                30
rbp            0x7f8e680be790      0x7f8e680be790
rsp            0x7f8e680be730      0x7f8e680be730
r8             0x8                 8
r9             0x1ffff             131071
r10            0x0                 0
r11            0x3                 3
r12            0x7f8e4dc81000      140249167040512
r13            0x55e3fadc3b08      94437654674184
r14            0x7f8e4defcff0      140249169645552
r15            0x7f8e4dc81000      140249167040512
rip            0x7f8ebb60da81      0x7f8ebb60da81 <_chash_init+465>
eflags         0x10206             [ PF IF RF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
gs             0x0                 0
fs_base        0x7f8e6810d680      140249608017536
gs_base        0x0                 0

(gdb)   p/x hash
$2 = 0x55e3fadc3b00
(gdb)   p/x hash->table
$3 = 0x7f8e4dc81000
(gdb)   p/x i
$4 = 0xf800
(gdb)   p sizeof(hash_line_t)
$5 = 16
(gdb)   p/x &hash->table[i]
$6 = 0x7f8e4dd79000

```

1. 从 _siginfo中得知， si_code = 1 = SEGV_MAPERR， 表示 CPU 访问 0x7f8e4dd79000 时，该地址没有有效内存映射。

2. 故障指令：  mov %r14,(%r12,%rax,1) 意思是 *(uint64_t *)(r12 + rax) = r14
  r12 = 0x7f8e4dc81000
  rax = 0x00000000000f8000
  r14 = 0x7f8e4defcff0

  r12 + rax  = 0x7f8e4dc81000  + 0x00000000000f8000  = 0x7f8e4dd79000 刚好等于 si_addr = 0x7f8e4dd79000
可以确认 SIGSEGV 就是这条 mov 指令向 0x7f8e4dd79000 写入时触发的。

3. 从gdb中得到
  hash->table        = 0x7f8e4dc81000
  i                  = 0xf800 = 63488
  sizeof(hash_line_t)= 16 = 0x10

&hash->table[i] = hash->table + i * sizeof(hash_line_t)
                = 0x7f8e4dc81000 + 0xf800 * 0x10
                = 0x7f8e4dc81000 + 0xf8000
                = 0x7f8e4dd79000


从代码上看,这是一个遍历，i 从 0开始到 bucket_size ,为什么到了 63488就不行了呢？

```
        for (i = 0; i < bucket_size; i++) {
//部分省略
                line[i].lock = lock_storage + (size_t)i * lock_size;

 //部分省略
        }

```

查看table，发现table[0] 也不可访问。 推断是遍历过程中，这块内存被unmap了。

```
(gdb) p/x hash->table
$9 = 0x7f8e4dc81000
(gdb) x/gx 0x7f8e4dc81000
0x7f8e4dc81000:	Cannot access memory at address 0x7f8e4dc81000
(gdb) p/x hash->table[0] 
Cannot access memory at address 0x7f8e4dc81000
(gdb) p/x hash->table[63488]
Cannot access memory at address 0x7f8e4dd79000

```


## 三、柳暗花明，转机到来

经过很多次的复现crash，分析，修改尝试。再复现crash，分析，修改尝试。
觉得只打印munmap，无法知道这个内存的由来。我们想要看看它什么时候被map的，什么时候被unmap的完整时间线。
于是增加了mmap的信息日志打印。

再次复现后，

```
(gdb) bt
#0  __memset_avx2_unaligned_erms () at ../sysdeps/x86_64/multiarch/memset-vec-unaligned-erms.S:326
#1  0x00005617e9e50e7b in _op_alloc_cache (op=op@entry=0x561823502bb0) at ./core/objectpool.c:154
#2  0x00005617e9e50d0d in objectpool_alloc (objsize=objsize@entry=2080, incsize=incsize@entry=500, locked=locked@entry=0) at ./core/objectpool.c:468

(gdb)   frame 1
#1  0x00005617e9e50e7b in _op_alloc_cache (op=op@entry=0x561823502bb0) at ./core/objectpool.c:154
(gdb)   p/x cache
$3 = 0x561823502ca0
(gdb)   p/x cache->pool
$4 = 0x7f0f7e674010        //pool的起始地址

(gdb)   p/x op
$5 = 0x561823502bb0
(gdb)   p/x op->objsize
$6 = 0x820
(gdb)   p/x op->nodesize
$7 = 0x848
(gdb)   p op->incsize
$8 = 500
(gdb)   p i
$9 = 193
(gdb)   p/x node
$10 = 0x7f0f7e6d7e58

(gdb)   p/x (char *)cache->pool + op->nodesize * op->incsize
$12 = 0x7f0f7e776cb0
(gdb)   p/x (char *)node + op->nodesize
$13 = 0x7f0f7e6d86a0                    //出问题时访问的地址

(gdb) x/gx 0x7f0f7e6d86a0
0x7f0f7e6d86a0:	Cannot access memory at address 0x7f0f7e6d86a0
(gdb) x/gx 0x7f0f7e6d7e58
0x7f0f7e6d7e58:	Cannot access memory at address 0x7f0f7e6d7e58
(gdb) x/gx 0x7f0f7e674010
0x7f0f7e674010:	Cannot access memory at address 0x7f0f7e674010

```

memset在操作 0x7f0f7e6d86a0 这个地址时发现内存没有映射，出错了。 
经检查 pool从起始位置 0x7f0f7e674010 也无法访问。 象是整块内存被unmap了。
0x7f0f7e674010 对应的 mmap addr时 0x7f0f7e674000

查看系统日志。

```
$ grep -Er 0x7f0f7e674000 messages
Aug 17 16:44:15 FortiWeb kern.info kernel: [221727.204921] mmap: PROXY_MMAP_RESULT seq=476016 tgid=3697 tid=3782 hint=0x0 len=0x103000 ret=0x7f0f7e674000 err=0 ip=0x7f0fdceb778c sp=0x7f0f94de85e8 ns=221727199929465 dur_ns=3604
Aug 17 16:44:15 FortiWeb kern.info kernel: [221727.205028] mmap: PROXY_MMAP_RESULT seq=476017 tgid=3697 tid=3697 hint=0x0 len=0x101000 ret=0x7f0f7e674000 err=0 ip=0x7f0fdceb778c sp=0x7ffef8113518 ns=221727200036369 dur_ns=19175
Aug 17 16:44:15 FortiWeb kern.info kernel: [221727.205164] mmap: PROXY_MUNMAP_ENTER seq=476019 tgid=3697 tid=3697 addr=0x7f0f7e674000 len=0x101000 ip=0x7f0fdceb7d97 sp=0x7ffef81135f8 ns=221727200172966
Aug 17 16:44:15 FortiWeb kern.info kernel: [221727.205208] mmap: PROXY_MUNMAP_RESULT seq=476019 tgid=3697 tid=3697 addr=0x7f0f7e674000 len=0x101000 ret=0 ip=0x7f0fdceb7d97 sp=0x7ffef81135f8 nr=2 raw_nr=16 stop=invalid_bp ns=221727200216463 dur_ns=43497

```

配合 gdb信息

```
(gdb) info thread
  Id   Target Id         Frame 
* 1    LWP 3782          __memset_avx2_unaligned_erms () at ../sysdeps/x86_64/multiarch/memset-vec-unaligned-erms.S:326
  24   LWP 3697          0x00007f0fdceb7d97 in __GI_munmap () at ../sysdeps/unix/syscall-template.S:117
```

我们发现在0.1ms左右 先是线程3782执行mmap，得到0x7f0f7e674000 。然后主线程3697执行mmap，也得到3697。
很快0.1ms左右主线程3697执行unmap，将0x7f0f7e674000内存释放，然后用户态线程3782就在执行memset时因为内存未映射crash了。

通过这些信息，我们发现问题出在内核 ，两次mmap返回的是同一个地址，导致了问题的发生。
至此，我们重新将目光转回到内核。还记得我说的一个伏笔么？当时一直认为是进程代码有问题出现的UAF，导致耗费了大量精力去分析proxyd_daemon代码，却一无所获。

## 四、 水落石出

查看内核 vma相关代码，发现里面应用了一个古早的产品patch，因为linux 6.1 和 linux 6.18 的vma代码差异，在应用中改动错误。
但这个错误，不影响内核稳定，而移植时只关注了内核是否稳定。没有测试应用程序，所以没有发现。
这个错误导致了两次mmap返回相同地址的问题。

```
@@ -2677,7 +2677,7 @@ static unsigned long __mmap_region(struct file *file, unsigned long addr,
 
        __mmap_complete(&map, vma);
 
-       return addr;
+       return vma->vm_start;
 
        /* Accounting was done by __mmap_prepare(). */
 unacct_error:

```

假如第一次匿名mmap，内核创建了一个VMA1 ，对应内存范围 [0x200000,0x300000) ，addr1= **0x200000** len1=0x100000, vma->vm_start= **0x200000** ,vma->vm_end=0x300000；
第二次匿名mmap，内核新增范围 [0x300000,0x400000) ,因为这两块内存相邻且属性相同，那么内核就会将它们合并。合并后VMA1对应 [0x200000, 0x400000);
此时 addr2= **0x300000** , 但是vma->vm_start= **0x200000** ,vma->vm_end=0x400000;

可以看出当vma内存合并后，第二次mmap 和 第一次 mmap 返回的 vm_start是 同一个 0x200000， 这就导致了问题的发生。

## 五、后记

进程的UAF的假象，导致在应用进程上耗费了大量的时间。后续应该吸取教训，避免类似问题发生。

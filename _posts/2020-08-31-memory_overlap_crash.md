---
title: "[Crash Analysis] Crash issue caused by slab memory overlap."
date: 2020-08-31
---

公司设备升级新版本后，出现了crash，描述"general protection fault: 0000 [#1] SMP "，初步看来是访问了异常地址。

异常栈如下：
```bash
PID: 945    TASK: ffff880079231f60  CPU: 0   COMMAND: "config_t/0"
 #0 [ffff88001037f9d0] machine_kexec at ffffffff81059bab
 #1 [ffff88001037fa30] __crash_kexec at ffffffff81105812
 #2 [ffff88001037fb00] crash_kexec at ffffffff81105900
 #3 [ffff88001037fb18] oops_end at ffffffff8168fdc8
 #4 [ffff88001037fb40] die at ffffffff8102e93b
 #5 [ffff88001037fb70] do_general_protection at ffffffff8168f6be
 #6 [ffff88001037fba0] general_protection at ffffffff8168ef68
    [exception RIP: _T_StatsCopy+71]
    RIP: ffffffffa051a117  RSP: ffff88001037fc58  RFLAGS: 00010297
    RAX: ffff88007fc00000  RBX: ffffffff81ae6340  RCX: 029d5fcc029d5fcc
    RDX: 0000000000000000  RSI: 0000000000000001  RDI: 0000000000000000
    RBP: ffff88001037fca0   R8: ffffffff81ae6340   R9: 000060ff7f232f90
    R10: 00000000000074a8  R11: 0000000000032c89  R12: ffff880002bdb3d0
    R13: 0000000000000001  R14: ffffc90000448020  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88001037fca8] _T_StructCopy at ffffffffa051adf2 [testmod]
 #8 [ffff88001037fcf8] T_StructCopy at ffffffffa0519303 [testmod]
 #9 [ffff88001037fd08] T_ConfFunc at ffffffffa0515994 [testmod]
#10 [ffff88001037fea0] T_ConfThread at ffffffffa0594625 [testmod]
#11 [ffff88001037fec8] kthread at ffffffff810b0a4f
#12 [ffff88001037ff50] ret_from_fork at ffffffff81697518
```

从crash位置指令看是访问Src struct结构的Stat指针，值为 029d5fcc029d5fcc，这是直接导致问题的原因。那么为什么会得到错误的值呢？

从寄存器 R12中找到 Src struct指针：ffff880002bdb3d0；
从堆栈中找到Dst struct的指针：        ffff880002bdb428。
因为Src struct和Dst struct都是 slab kmalloc-192 中申请的，内存块的大小是192字节。而Dst struct和Src struct的差距只有58个字节。也就是说Dst struct和Src struct产生了部分重叠。

Src struct和Dst struct指向的地址空间内容
```bash
crash> rd ffff880002bdb3c0 32
ffff880002bdb3c0:  000000004a41434b 00000000000000a4   KCAJ............
ffff880002bdb3d0:  0000000502907300 0000017400000000   .s..........t...      //ffff880002bdb3d0 Src
ffff880002bdb3e0:  0263026300000000 000005011e010101   ....c.c.........
ffff880002bdb3f0:  029d5fcc029d5fcc ffff88001dcf27b8   ._..._...'......
ffff880002bdb400:  ffffc90000448020 0000000000000000    .D.............
ffff880002bdb410:  0000000000000000 000000004a41434b   ........KCAJ....
ffff880002bdb420:  00000000000000a4 0000000502907300   .........s......    //ffff880002bdb428 Dst
ffff880002bdb430:  0000017400000000 0263026300000000   ....t.......c.c.
ffff880002bdb440:  000005011e010101 029d5fcc029d5fcc   ........._..._..    //ffff880002bdb448 Stats内容029d5fcc029d5fcc
ffff880002bdb450:  ffff88001dcf27b8 ffffc90000448020   .'...... .D.....
ffff880002bdb460:  0000000000000000 0000000000000000   ................
ffff880002bdb470:  000000004a41434b 00000000000000a4   KCAJ............
//正常应该在这里结束，因为copy 重叠，将Src的内容覆写。
ffff880002bdb480:  0000000502907300 0000017400000000   .s..........t...
ffff880002bdb490:  0263026300000000 000005011e010101   ....c.c.........
ffff880002bdb4a0:  000060ff7f232f90 000060ff7f232c8c   ./#..`...,#..`..
ffff880002bdb4b0:  ffffc90000448020 0000000058494e47    .D.....GNIX....
```

将Src struct拷贝到Dst struct时，因为内存有重叠。导致Dst struct和Src struct重叠的部分被写坏了，最终导致访问这部分内容时，异常crash。

那么Dst struct指针为什么会异常呢？Dst struct结构也是kmem_cache_alloc创建的，是从slab上申请的。造成申请出的指针异常，只有可能是上次释放时传入的指针错误，导致这个错误的指针被放入到slab的freelist链上。下次申请struct的时候得到了这个错误的指针，然后进行操作导致问题发生。

查看相关代码，发现最近有个修改存在一个并发问题，导致struct内存释放时，又会调用call_rcu将struct挂到rcu的nxtlist上，等到静默期过后，执行struct的rcu func，将struct挂到一个hash链表上。

但是因为struct已经被释放了，在某个时刻被申请后整块内存置0，也就是rcu_head的 next和func都为0。

静默期后，在__rcu_reclaim中执行struct的callback处理， 因为func为0  < 4096，被当作要kfree的结构，通过rcu_head指针-(func值代表的偏移)来得到要释放结构的内存块指针，然后执行kfree。


```c
static inline bool __rcu_reclaim(char *rn, struct rcu_head *head)
{
 unsigned long offset = (unsigned long)head->func;

 if (__is_kfree_rcu_offset(offset)) {
  RCU_TRACE(trace_rcu_invoke_kfree_callback(rn, head, offset));
  kfree((void *)head - offset);
  return 1;
 } else {
  RCU_TRACE(trace_rcu_invoke_callback(rn, head));
  head->func(head);
  return 0;
 }
}
```

导致将struct的 rcu_head指针当作了内存块的指针进行kfree释放操作，该指针放到了slab的freelist链表中，等到下次申请struct内存时，得到了这个错误的指针。

Dst struct的指针正好在Src struct的rcu_head位置。

后记：
1. 从问题中我们发现kfree也可以释放kmem_cache_alloc申请的内存。
2. kfree和kmem_cache_free传入错误的指针，内核并不会第一时间发现，只是在后面会造成不可预知的错误。

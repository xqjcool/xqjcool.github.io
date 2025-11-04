---
title: Crash caused by an invalid address access when the timer was executed
date: 2025-11-04
---

# 定时器执行时因为异常地址访问导致的crash

## 1. 问题现象

测试报告说遇到了一个crash问题，取core文件查看。

```bash
[128629.828210] BUG: unable to handle page fault for address: 0000000179481008
[128629.828340] #PF: supervisor write access in kernel mode
[128629.828385] #PF: error_code(0x0002) - not-present page
[128629.828429] PGD 0 P4D 0 
[128629.828456] Oops: 0002 [#1] SMP NOPTI
[128629.828491] CPU: 1 PID: 0 Comm: swapper/1 Kdump: loaded Tainted: G        W  O       6.1 #1
[128629.828564] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
[128629.828669] RIP: 0010:run_timer_softirq+0x645/0x700
[128629.828716] Code: 63 d3 00 48 c7 43 18 00 00 00 00 4e 8b 7c ec 40 4d 85 ff 74 a0 4c 89 7b 18 66 90 49 8b 07 49 8b 4f 08 48 89 01 48 85 c0 74 04 <48> 89 48 08 49 c7 47 08 00 00 00 00 48 b8 22 01 00 00 00 00 ad de
[128629.828863] RSP: 0018:ffffc900002f0e90 EFLAGS: 00010006
[128629.828915] RAX: 0000000179481000 RBX: ffff888237d1db40 RCX: ffffc900002f0ed8
[128629.828976] RDX: 0000000000000000 RSI: 0000000107a62808 RDI: 0000000107a62808
[128629.829059] RBP: ffffc900002f0f60 R08: 0000000020f4c501 R09: 0000000000000000
[128629.829124] R10: ffff888237d1dbc0 R11: 0000000000000080 R12: 0000000000000040
[128629.829185] R13: 0000000000000001 R14: 0000000107a62800 R15: ffff888100a9b670
[128629.829247] FS:  0000000000000000(0000) GS:ffff888237d00000(0000) knlGS:0000000000000000
[128629.829314] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[128629.829372] CR2: 0000000179481008 CR3: 000000010ae68000 CR4: 00000000000006e0
[128629.829447] Call Trace:
[128629.829473]  <IRQ>
[128629.829496]  ? __die_body+0x82/0x130
[128629.829533]  ? page_fault_oops+0x364/0x450
[128629.829577]  ? kernelmode_fixup_or_oops+0x11d/0x190
[128629.829620]  ? update_load_avg+0x1c4/0x290
[128629.829659]  ? enqueue_task_fair+0x334/0x660
[128629.829698]  ? __bad_area_nosemaphore+0x53/0x300
[128629.829739]  ? load_balance+0x5e4/0x1f10
[128629.829780]  ? bad_area_nosemaphore+0x16/0x20
[128629.829820]  ? do_user_addr_fault+0x499/0x6b0
[128629.829865]  ? ct_nmi_enter+0x94/0xc0
[128629.829901]  ? exc_page_fault+0x59/0xc0
[128629.829938]  ? asm_exc_page_fault+0x27/0x30
[128629.829977]  ? run_timer_softirq+0x645/0x700
[128629.830017]  ? update_process_times+0x6c/0x80
[128629.830057]  ? lapic_next_event+0x1f/0x30
[128629.830095]  __do_softirq+0x14a/0x390
[128629.830130]  irq_exit_rcu+0x71/0xa0
[128629.830166]  sysvec_apic_timer_interrupt+0x7b/0x90
[128629.830209]  </IRQ>

```

异常地址访问发生在timer中断处理中，大概率是某个timer的使用方法不对导致。

## 2. 初步分析

使用crash工具体调试coredump文件

```bash
crash> dis -l run_timer_softirq+0x645
/root/kernel/linux-6.1.62/include/linux/list.h: 886
0xffffffff8036d0b5 <run_timer_softirq+1605>:    mov    %rcx,0x8(%rax)
```

这是__hlist_del删除节点操作

```c
static inline void __hlist_del(struct hlist_node *n)
{
        struct hlist_node *next = n->next;
        struct hlist_node **pprev = n->pprev;

        WRITE_ONCE(*pprev, next);
        if (next)
                WRITE_ONCE(next->pprev, pprev);    //将pprev赋予next->pprev时，因为next异常导致
}
```

我们查看问题发生位置前后的汇编代码

```bash
0xffffffff8036d08e <run_timer_softirq+1566>:    movq   $0x0,0x18(%rbx)
0xffffffff8036d096 <run_timer_softirq+1574>:    mov    0x40(%rsp,%r13,8),%r15
0xffffffff8036d09b <run_timer_softirq+1579>:    test   %r15,%r15
0xffffffff8036d09e <run_timer_softirq+1582>:    je     0xffffffff8036d040 <run_timer_softirq+1488>
0xffffffff8036d0a0 <run_timer_softirq+1584>:    mov    %r15,0x18(%rbx)
0xffffffff8036d0a4 <run_timer_softirq+1588>:    xchg   %ax,%ax
0xffffffff8036d0a6 <run_timer_softirq+1590>:    mov    (%r15),%rax    //next = timer_list.entry.next
0xffffffff8036d0a9 <run_timer_softirq+1593>:    mov    0x8(%r15),%rcx
0xffffffff8036d0ad <run_timer_softirq+1597>:    mov    %rax,(%rcx)
0xffffffff8036d0b0 <run_timer_softirq+1600>:    test   %rax,%rax
0xffffffff8036d0b3 <run_timer_softirq+1603>:    je     0xffffffff8036d0b9 <run_timer_softirq+1609>
0xffffffff8036d0b5 <run_timer_softirq+1605>:    mov    %rcx,0x8(%rax)    //问题发生位置
0xffffffff8036d0b9 <run_timer_softirq+1609>:    movq   $0x0,0x8(%r15)
0xffffffff8036d0c1 <run_timer_softirq+1617>:    movabs $0xdead000000000122,%rax
0xffffffff8036d0cb <run_timer_softirq+1627>:    mov    %rax,(%r15)
0xffffffff8036d0ce <run_timer_softirq+1630>:    mov    0x18(%r15),%r12
0xffffffff8036d0d2 <run_timer_softirq+1634>:    testb  $0x20,0x22(%r15)
0xffffffff8036d0d7 <run_timer_softirq+1639>:    je     0xffffffff8036d070 <run_timer_softirq+1536>
0xffffffff8036d0d9 <run_timer_softirq+1641>:    mov    %rbx,%rdi
0xffffffff8036d0dc <run_timer_softirq+1644>:    call   0xffffffff810a3340 <_raw_spin_unlock>
0xffffffff8036d0e1 <run_timer_softirq+1649>:    mov    %r15,%rdi

```

这部分对应 函数 expire_timers。

```c
static void expire_timers(struct timer_base *base, struct hlist_head *head)
{
        /*
         * This value is required only for tracing. base->clk was
         * incremented directly before expire_timers was called. But expiry
         * is related to the old base->clk value.
         */
        unsigned long baseclk = base->clk - 1;

        while (!hlist_empty(head)) {
                struct timer_list *timer;
                void (*fn)(struct timer_list *);

                timer = hlist_entry(head->first, struct timer_list, entry);

                base->running_timer = timer;
                detach_timer(timer, true);    //调用__list_del 最终触发异常

                fn = timer->function;

                if (timer->flags & TIMER_IRQSAFE) {
                        raw_spin_unlock(&base->lock);
                        call_timer_fn(timer, fn, baseclk);
                        raw_spin_lock(&base->lock);
                        base->running_timer = NULL;
                } else {
                        raw_spin_unlock_irq(&base->lock);
                        call_timer_fn(timer, fn, baseclk);
                        raw_spin_lock_irq(&base->lock);
                        base->running_timer = NULL;
                        timer_sync_wait_running(base);
                }
        }
}

```

结合以上内容， 可以得知： r15就是 hlist_node， 也是 timer_list

```bash
crash> timer_list -x ffff888100a9b670 
struct timer_list {
  entry = {
    next = 0x179481000,    //值异常，导致执行 next->pprev时 触发异常地址访问
    pprev = 0xffffc900002f0ed8
  },
  expires = 0xffffea0007373540,
  function = 0x100000000000,
  flags = 0xcdcd5000
}

```

很明显这个timer_list的多个字段都不正常。查看内存，发现是已经释放的内存。这是timer使用中较常见的错误， timer所在内存已经被释放了，但timer还在队列中。等到timer中断执行到对应timer时，就因为异常内存访问导致crash。

```bash
crash> kmem ffff888100a9b670
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff8881009fe700      512          4        32      2     8k  sgpool-16
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea000402a680  ffff888100a9a000     0     16          2    14
  FREE / [ALLOCATED]
   ffff888100a9b600  (cpu 1 cache)

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea000402a6c0 100a9b000        200000000        0  0 8000000000000000
```


因为function函数也被覆盖了，导致无法直接判断是那个timer。

## 3. 调试复现

这种情况下，只看cordump时不太能找到问题timer的。我们需要开启调试选项，并对问题进行复现。

```bash
+CONFIG_DEBUG_OBJECTS=y
+CONFIG_DEBUG_OBJECTS_FREE=y
+CONFIG_DEBUG_OBJECTS_TIMERS=y
```

.config文件添加以上CONFIG选项，重新编译image，在问题设备上进行复现。


## 4. 后记

遇到类似问题，大多都是相关内存释放，但是没有删除timer导致的。如果能稳定复现，开启相关选项后，就能很快捕捉到问题timer了。
timer在内核中非常普及，任何内核和模块开发都会经常用到，但只有部分人能够熟练正确使用。



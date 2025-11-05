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
再次复现后，查看日志

```bash
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552402] ------------[ cut here ]------------
Nov  4 13:59:19 VM kern.err kernel: [ 1369.552407] ODEBUG: free active (active state 0) object type: timer_list hint: ddos_log_expire+0x0/0x500
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552672] WARNING: CPU: 1 PID: 1598 at lib/debugobjects.c:517 debug_check_no_obj_freed+0x1db/0x330
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552680] Modules linked in: ha(O) log_cfg(O) vtb(O) fs_miglog(O) miglog(O) adc_nl_ipc_k(O) bridge_mac(O) ti_bridge(O) infodmem(O)
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552693] CPU: 1 PID: 1598 Comm: fnginxctld Kdump: loaded Tainted: G           O       6.1 #71
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552697] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552699] RIP: 0010:debug_check_no_obj_freed+0x1db/0x330
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552703] Code: 89 05 81 53 a0 01 8b 41 10 8b 49 14 48 8b 14 c5 20 37 90 81 4d 8b 07 48 c7 c7 28 16 7f 81 48 c7 c6 41 0d 80 81 e8 f5 f9 a8 ff <0f> 0b ff 05 bd 93 64 01 4c 8b 7d d0 48 8b 45 c0 4c 8b 58 30 4d 85
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552706] RSP: 0000:ffffc90002877d50 EFLAGS: 00010246
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552710] RAX: ef5fbc460b011600 RBX: ffff888100aa7600 RCX: 0000000000000027
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552712] RDX: ffffc90002877c40 RSI: 00000000ffffdfff RDI: ffff888237d1d7e8
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552714] RBP: ffffc90002877db0 R08: 0000000000001fff R09: ffffffff81c54650
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552716] R10: 0000000000005ffd R11: 0000000000000004 R12: ffff888100aa7470
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552718] R13: ffff8881034bae38 R14: ffff888100aa7400 R15: ffffffff818c76c0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552721] FS:  00007fa68dcd9800(0000) GS:ffff888237d00000(0000) knlGS:0000000000000000
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552723] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552726] CR2: 00007fbe6d22a018 CR3: 00000001096fe000 CR4: 00000000000006e0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552760] Call Trace:
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552762]  <TASK>
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552764]  ? __warn+0x197/0x270
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552769]  ? debug_check_no_obj_freed+0x1db/0x330
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552773]  ? report_bug+0x1af/0x240
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552778]  ? console_unlock+0x27b/0x2d0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552782]  ? handle_bug+0x46/0x80
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552785]  ? exc_invalid_op+0x1b/0x50
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552788]  ? asm_exc_invalid_op+0x1b/0x20
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552793]  ? debug_check_no_obj_freed+0x1db/0x330
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552797]  kmem_cache_free+0x160/0x290
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552812]  ? ddos_free_rcu+0x1c/0x30
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552816]  ddos_free_rcu+0x1c/0x30
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552820]  rcu_core+0x2f3/0x900
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552825]  rcu_core_si+0xe/0x20
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552828]  __do_softirq+0x14a/0x390
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552832]  irq_exit_rcu+0x71/0xa0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552837]  sysvec_apic_timer_interrupt+0x3d/0x90
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552841]  asm_sysvec_apic_timer_interrupt+0x1b/0x20
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552845] RIP: 0033:0x7fa68f238893
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552848] Code: 00 00 00 76 2a 48 8d 50 40 48 83 e2 f0 66 2e 0f 1f 84 00 00 00 00 00 0f 29 02 0f 29 42 10 0f 29 42 20 0f 29 42 30 48 83 ea c0 <48> 39 fa 72 e8 0f 11 07 0f 11 47 10 0f 11 47 20 0f 11 47 30 c3 0f
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552850] RSP: 002b:00007ffc6316b318 EFLAGS: 00000207
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552852] RAX: 000055d1e080ac20 RBX: 000000000000000c RCX: 000055d1e120f770
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552854] RDX: 000055d1e08be1a0 RSI: 0000000000000000 RDI: 000055d1e090abe0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552856] RBP: 000055d1e080ac20 R08: 0000000000015890 R09: 0000000000000000
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552858] R10: 000055d1e07f5820 R11: 0000000000000202 R12: 000055d1e0809728
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552860] R13: 000055d1e080a3c0 R14: 000055d1e092944c R15: 00007ffc6326b3c0
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552862]  </TASK>
Nov  4 13:59:19 VM kern.warn kernel: [ 1369.552863] ---[ end trace 0000000000000000 ]---

```

从以上Call Trace可以看到，问题timer是 ddos_log_expire.

随后查看对应代码，梳理 dos_vs_log_expire 的相关使用逻辑，发现存在 timer还在队列中，但是相关内存结构被释放的问题。


## 4. 后记

遇到类似问题，大多都是相关内存释放，但是没有删除timer导致的。如果能稳定复现，开启相关选项后，就能很快捕捉到问题timer了。这极大的提高了问题定位效率。
timer在内核中非常普及，任何内核和模块开发都会经常用到，但只有部分人能够熟练正确使用。



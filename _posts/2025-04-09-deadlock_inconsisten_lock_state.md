---
title: "Deadlock caused by locking in process context and interrupt context."
date: 2025-04-09
---

# 进程上下文加锁和中断上下文加锁导致的死锁
在测试环境验证问题时，触发了一个deadlock异常。

## 异常日志 WARNING: inconsistent lock state

因为是调试验证，所以打开了 CONFIG_PROVE_LOCKING 选项，系统捕获到并打印了以下日志。这段详细的告警信息，帮助迅速定位到错误原因。

```bash
Apr  9 16:20:26 kern.warn kernel: [   65.630201] ================================
Apr  9 16:20:26 kern.warn kernel: [   65.630294] WARNING: inconsistent lock state
Apr  9 16:20:26 kern.warn kernel: [   65.630416] 6.1 #4 Tainted: G           O      
Apr  9 16:20:26 kern.warn kernel: [   65.630506] --------------------------------
Apr  9 16:20:26 kern.warn kernel: [   65.630618] inconsistent {SOFTIRQ-ON-W} -> {IN-SOFTIRQ-W} usage.
Apr  9 16:20:26 kern.warn kernel: [   65.630771] swapper/1/0 [HC0[0]:SC1[1]:HE1:SE0] takes:
Apr  9 16:20:26 kern.warn kernel: [   65.630901] ffff88810d520018 (&net->ipv4.fib_statistics->lock){+.?.}-{2:2}, at: net_event_callback+0x373/0x490
Apr  9 16:20:26 kern.warn kernel: [   65.631135] {SOFTIRQ-ON-W} state was registered at:
Apr  9 16:20:26 kern.warn kernel: [   65.631257]   lock_acquire+0xc2/0x290
Apr  9 16:20:26 kern.warn kernel: [   65.631369]   _raw_spin_lock+0x2f/0x50
Apr  9 16:20:26 kern.warn kernel: [   65.631473]   fib_stat_rcv_proc+0x5b8/0x790
Apr  9 16:20:26 kern.warn kernel: [   65.631589]   netlink_rcv_skb+0x57/0x110
Apr  9 16:20:26 kern.warn kernel: [   65.631694]   nl_gateway_input+0x2a/0x40
Apr  9 16:20:26 kern.warn kernel: [   65.631803]   netlink_unicast+0x1ab/0x290
Apr  9 16:20:26 kern.warn kernel: [   65.631908]   netlink_sendmsg+0x22d/0x480
Apr  9 16:20:26 kern.warn kernel: [   65.631992]   ____sys_sendmsg+0x230/0x260
Apr  9 16:20:26 kern.warn kernel: [   65.632078]   ___sys_sendmsg+0x96/0xd0
Apr  9 16:20:26 kern.warn kernel: [   65.632155]   __sys_sendmsg+0x7d/0xd0
Apr  9 16:20:26 kern.warn kernel: [   65.632231]   __x64_sys_sendmsg+0x1d/0x30
Apr  9 16:20:26 kern.warn kernel: [   65.632312]   do_syscall_64+0x37/0x90
Apr  9 16:20:26 kern.warn kernel: [   65.632392]   entry_SYSCALL_64_after_hwframe+0x64/0xce
Apr  9 16:20:26 kern.warn kernel: [   65.632492] irq event stamp: 747160
Apr  9 16:20:26 kern.warn kernel: [   65.632562] hardirqs last  enabled at (747160): [<ffffffff80c3f954>] seqcount_lockdep_reader_access+0x84/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.632768] hardirqs last disabled at (747159): [<ffffffff80c3f926>] seqcount_lockdep_reader_access+0x56/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.632946] softirqs last  enabled at (747080): [<ffffffff802836aa>] __irq_exit_rcu+0x6a/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.633115] softirqs last disabled at (747137): [<ffffffff802836aa>] __irq_exit_rcu+0x6a/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.633273] 
Apr  9 16:20:26 kern.warn kernel: [   65.633273] other info that might help us debug this:
Apr  9 16:20:26 kern.warn kernel: [   65.633393]  Possible unsafe locking scenario:
Apr  9 16:20:26 kern.warn kernel: [   65.633393] 
Apr  9 16:20:26 kern.warn kernel: [   65.633513]        CPU0
Apr  9 16:20:26 kern.warn kernel: [   65.633568]        ----
Apr  9 16:20:26 kern.warn kernel: [   65.633623]   lock(&net->ipv4.fib_statistics->lock);
Apr  9 16:20:26 kern.warn kernel: [   65.633712]   <Interrupt>
Apr  9 16:20:26 kern.warn kernel: [   65.633769]     lock(&net->ipv4.fib_statistics->lock);
Apr  9 16:20:26 kern.warn kernel: [   65.633861] 
Apr  9 16:20:26 kern.warn kernel: [   65.633861]  *** DEADLOCK ***
Apr  9 16:20:26 kern.warn kernel: [   65.633861] 
Apr  9 16:20:26 kern.warn kernel: [   65.633977] 3 locks held by swapper/1/0:
Apr  9 16:20:26 kern.warn kernel: [   65.634056]  #0: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: netif_receive_skb_list_internal+0xbb/0x3a0
Apr  9 16:20:26 kern.warn kernel: [   65.634244]  #1: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: atomic_notifier_call_chain+0x29/0x170
Apr  9 16:20:26 kern.warn kernel: [   65.634422]  #2: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: net_event_callback+0x12a/0x490
Apr  9 16:20:26 kern.warn kernel: [   65.634582] 
Apr  9 16:20:26 kern.warn kernel: [   65.634582] stack backtrace:
Apr  9 16:20:26 kern.warn kernel: [   65.634671] CPU: 1 PID: 0 Comm: swapper/1 Kdump: loaded Tainted: G           O       6.1 #4
Apr  9 16:20:26 kern.warn kernel: [   65.634824] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
Apr  9 16:20:26 kern.warn kernel: [   65.635010] Call Trace:
Apr  9 16:20:26 kern.warn kernel: [   65.635067]  <IRQ>
Apr  9 16:20:26 kern.warn kernel: [   65.635118]  dump_stack_lvl+0x60/0x7e
Apr  9 16:20:26 kern.warn kernel: [   65.635198]  dump_stack+0x10/0x16
Apr  9 16:20:26 kern.warn kernel: [   65.635271]  print_usage_bug.part.0+0x1a9/0x1c0
Apr  9 16:20:26 kern.warn kernel: [   65.635361]  mark_lock.cold+0x31/0xe1
Apr  9 16:20:26 kern.warn kernel: [   65.635437]  ? rcu_read_lock_held+0x42/0x50
Apr  9 16:20:26 kern.warn kernel: [   65.635526]  ? find_held_lock+0x31/0x90
Apr  9 16:20:26 kern.warn kernel: [   65.635605]  ? __inet_dev_addr_type+0x15e/0x340
Apr  9 16:20:26 kern.warn kernel: [   65.635707]  ? lock_release+0xce/0x290
Apr  9 16:20:26 kern.warn kernel: [   65.635787]  __lock_acquire+0x9b4/0x1e70
Apr  9 16:20:26 kern.warn kernel: [   65.635867]  ? __lock_acquire+0x38b/0x1e70
Apr  9 16:20:26 kern.warn kernel: [   65.635951]  lock_acquire+0xc2/0x290
Apr  9 16:20:26 kern.warn kernel: [   65.636031]  ? net_event_callback+0x373/0x490
Apr  9 16:20:26 kern.warn kernel: [   65.636113]  _raw_spin_lock+0x2f/0x50	//spin_lock(&net->ipv4.fib_statistics->lock)
Apr  9 16:20:26 kern.warn kernel: [   65.636189]  ? net_event_callback+0x373/0x490
Apr  9 16:20:26 kern.warn kernel: [   65.636268]  net_event_callback+0x373/0x490
Apr  9 16:20:26 kern.warn kernel: [   65.636346]  atomic_notifier_call_chain+0x84/0x170
Apr  9 16:20:26 kern.warn kernel: [   65.636442]  call_netevent_notifiers+0x1b/0x30
Apr  9 16:20:26 kern.warn kernel: [   65.636532]  __neigh_update+0x71c/0xda0
Apr  9 16:20:26 kern.warn kernel: [   65.636613]  neigh_event_ns+0x8f/0xc0
Apr  9 16:20:26 kern.warn kernel: [   65.636691]  arp_process+0xa4b/0xb50
Apr  9 16:20:26 kern.warn kernel: [   65.636767]  arp_rcv+0x142/0x320
Apr  9 16:20:26 kern.warn kernel: [   65.636837]  __netif_receive_skb_list_core+0x1ec/0x240
Apr  9 16:20:26 kern.warn kernel: [   65.636940]  netif_receive_skb_list_internal+0x1ca/0x3a0
Apr  9 16:20:26 kern.warn kernel: [   65.637044]  napi_complete_done+0x74/0x1b0
Apr  9 16:20:26 kern.warn kernel: [   65.637129]  vmxnet3_poll_rx_only+0x7b/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.637214]  __napi_poll.constprop.0+0x2e/0x190
Apr  9 16:20:26 kern.warn kernel: [   65.637306]  net_rx_action+0x150/0x290
Apr  9 16:20:26 kern.warn kernel: [   65.637387]  __do_softirq+0xd1/0x3ff
Apr  9 16:20:26 kern.warn kernel: [   65.637464]  __irq_exit_rcu+0x6a/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.637543]  irq_exit_rcu+0xe/0x20
Apr  9 16:20:26 kern.warn kernel: [   65.637618]  sysvec_apic_timer_interrupt+0x7c/0xa0
Apr  9 16:20:26 kern.warn kernel: [   65.637712]  </IRQ>

```

## 日志解析

### `inconsistent {SOFTIRQ-ON-W} -> {IN-SOFTIRQ-W} usage.`
- 左边 {SOFTIRQ-ON-W}：表示当前获取锁的代码是在 softirq 开启的上下文中，并且以写模式加锁（W）；
- 右边 {IN-SOFTIRQ-W}：表示当前上下文其实是 处于 SoftIRQ（软中断）中，并且也是以 写模式加锁
错误就在于，锁在“非软中断上下文中打开软中断状态”时被加锁，但同一个锁又在 SoftIRQ 上下文中被加锁，可能产生死锁。

### `swapper/1/0 [HC0[0]:SC1[1]:HE1:SE0] takes:`
表示当前锁是在 `swapper/1/`（即 CPU1 的 idle 进程）中上锁的。
此时的Lockdep 中断状态

| 标记 | 含义 | 值 |
| ------ | ------ | ------ |
| HC | 硬中断计数 (Hardirq count) | 0 |
| SC | 软中断计数 (Softirq count) | 1 |
| HE | 硬中断使能 (Hardirq enabled) | 1 |
| SE | 软中断使能 (Softirq enabled) | 0 |

综合起来就是：
当前 CPU（CPU1）正在执行 swapper idle task，正处于软中断上下文中（SoftIRQ count = 1），硬中断是启用的，软中断是禁用的。


### `ffff88810d520018 (&net->ipv4.fib_statistics->lock){+.?.}-{2:2}, at: net_event_callback+0x373/0x490`


在 net_event_callback 函数中，地址为 ffff88810d520018 的锁（即 net->ipv4.fib_statistics->lock）被加锁了，此时它已经处于加锁状态 {+.?.}，而它属于 Lockdep 的第 2 类锁，当前嵌套层数为 2（-{2:2}）。这个锁的使用可能引起不一致（尤其是和 SoftIRQ 的上下文切换有关）。

### `{SOFTIRQ-ON-W} state was registered at:`

这是进程上下文上锁的调用栈，可以看到是netlink收报处理过程中调用`fib_stat_rcv_proc`，对 `et->ipv4.fib_statistics->lock` 进行了加锁。

```bash
  lock_acquire+0xc2/0x290
  _raw_spin_lock+0x2f/0x50        //spin_lock(&net->ipv4.fib_statistics->lock)
  fib_stat_rcv_proc+0x5b8/0x790    //加锁函数
  netlink_rcv_skb+0x57/0x110
  nl_gateway_input+0x2a/0x40
  netlink_unicast+0x1ab/0x290
  netlink_sendmsg+0x22d/0x480
  ____sys_sendmsg+0x230/0x260
  ___sys_sendmsg+0x96/0xd0
  __sys_sendmsg+0x7d/0xd0
  __x64_sys_sendmsg+0x1d/0x30
  do_syscall_64+0x37/0x90
  entry_SYSCALL_64_after_hwframe+0x64/0xce
```
### 中断开启和关闭时间戳信息
和本次问题关系不大，如果有中断中调用shcedule错误的话，需要关注这一段。

```bash
 [   65.632492] irq event stamp: 747160
 [   65.632562] hardirqs last  enabled at (747160): [<ffffffff80c3f954>] seqcount_lockdep_reader_access+0x84/0xa0
 [   65.632768] hardirqs last disabled at (747159): [<ffffffff80c3f926>] seqcount_lockdep_reader_access+0x56/0xa0
 [   65.632946] softirqs last  enabled at (747080): [<ffffffff802836aa>] __irq_exit_rcu+0x6a/0xa0
 [   65.633115] softirqs last disabled at (747137): [<ffffffff802836aa>] __irq_exit_rcu+0x6a/0xa0
```

### 场景推演
系统推测到可能出现以下场景，并导致死锁。

```bash
[   65.633273] other info that might help us debug this:
[   65.633393]  Possible unsafe locking scenario:
[   65.633393] 
[   65.633513]        CPU0
[   65.633568]        ----
[   65.633623]   lock(&net->ipv4.fib_statistics->lock);
[   65.633712]   <Interrupt>
[   65.633769]     lock(&net->ipv4.fib_statistics->lock);
[   65.633861] 
[   65.633861]  *** DEADLOCK ***
```

### rcu_read锁持有情况
这部分与本次问题关系不大。

```bash
 [   65.633977] 3 locks held by swapper/1/0:
 [   65.634056]  #0: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: netif_receive_skb_list_internal+0xbb/0x3a0
 [   65.634244]  #1: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: atomic_notifier_call_chain+0x29/0x170
 [   65.634422]  #2: ffffffff81958a40 (rcu_read_lock){....}-{1:2}, at: net_event_callback+0x12a/0x490
```

### 中断栈信息
这里打印了中断栈调用信息，帮助更好的分析问题。

```bash
 [   65.634582] stack backtrace:
 [   65.634671] CPU: 1 PID: 0 Comm: swapper/1 Kdump: loaded Tainted: G           O       6.1 #4
 [   65.634824] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
 [   65.635010] Call Trace:
 [   65.635067]  <IRQ>
 [   65.635118]  dump_stack_lvl+0x60/0x7e
 [   65.635198]  dump_stack+0x10/0x16
 [   65.635271]  print_usage_bug.part.0+0x1a9/0x1c0
 [   65.635361]  mark_lock.cold+0x31/0xe1
 [   65.635437]  ? rcu_read_lock_held+0x42/0x50
 [   65.635526]  ? find_held_lock+0x31/0x90
 [   65.635605]  ? __inet_dev_addr_type+0x15e/0x340
 [   65.635707]  ? lock_release+0xce/0x290
 [   65.635787]  __lock_acquire+0x9b4/0x1e70
 [   65.635867]  ? __lock_acquire+0x38b/0x1e70
 [   65.635951]  lock_acquire+0xc2/0x290
 [   65.636031]  ? net_event_callback+0x373/0x490
 [   65.636113]  _raw_spin_lock+0x2f/0x50	        //spin_lock(&net->ipv4.fib_statistics->lock)
 [   65.636189]  ? net_event_callback+0x373/0x490
 [   65.636268]  net_event_callback+0x373/0x490      //加锁函数
 [   65.636346]  atomic_notifier_call_chain+0x84/0x170
 [   65.636442]  call_netevent_notifiers+0x1b/0x30
 [   65.636532]  __neigh_update+0x71c/0xda0
 [   65.636613]  neigh_event_ns+0x8f/0xc0
 [   65.636691]  arp_process+0xa4b/0xb50
 [   65.636767]  arp_rcv+0x142/0x320
 [   65.636837]  __netif_receive_skb_list_core+0x1ec/0x240
```

## 后记
可以看到系统告警打印还是非常详细的，开启 CONFIG_PROVE_LOCKING后，能够对 rcu_read_lock/unlock,read_lock/unlock,write_lock/unlock,spin_lock/unlock等锁进行跟踪并分析，帮助快速定位锁相关问题。
这个告警是在问题还未发生，但是存在异常可能时就提前告知，还是非常高效的。对于一些很复现的问题，我们就能在早期就发现并解决。

回到本次问题上，并不需要全部查看这么详细的日志。只要看下面三个地方即可。 


```bash
[   65.633513]        CPU0
[   65.633568]        ----
[   65.633623]   lock(&net->ipv4.fib_statistics->lock);
[   65.633712]   <Interrupt>
[   65.633769]     lock(&net->ipv4.fib_statistics->lock);
[   65.633861] 
[   65.633861]  *** DEADLOCK ***
```
表示进程栈和中断栈都对 `net->ipv4.fib_statistics->lock` 加锁导致死锁。

以及
```bash
{SOFTIRQ-ON-W} state was registered at:
  lock_acquire+0xc2/0x290
  _raw_spin_lock+0x2f/0x50
  fib_stat_rcv_proc+0x5b8/0x790
```
进程栈调用位置。

和 
```bash
 [   65.636113]  _raw_spin_lock+0x2f/0x50	//spin_lock(&net->ipv4.fib_statistics->lock)
 [   65.636189]  ? net_event_callback+0x373/0x490
 [   65.636268]  net_event_callback+0x373/0x490
```
中断栈调用位置。


很快就能知道， 因为 `net->ipv4.fib_statistics->lock` 在进程上下文和中断上下文都会加锁，所以我们需要使用 spin_lock_bh/spin_unlock_bh ，来避免嵌套导致死锁。


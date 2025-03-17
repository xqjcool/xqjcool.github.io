---
title: "[Crash Analysis] Cascade Error, Ultimately Identified as a Kernel BUG"
date: 2020-08-08
---

我们目前使用的是标准的CentOS 3.10.0-514.26.2版本。因为要做docker产品，为了测试设备可以支持多少个docker，我们使用脚本在设备上不断创建docker并运行我们的产品。

但是我们发现每次到了290多个docker后，很快系统就crash了。

crash栈大体上有两种：

```bash
[ 1076.347341] ------------[ cut here ]------------
[ 1076.361159] kernel BUG at kernel/timer.c:1105!
[ 1076.374470] invalid opcode: 0000 [#1] SMP
//。。。省略无关
[ 1076.638557] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G        W  OE  ------------ T 3.10.0-514.26.2.el7.x86_64 #1
[ 1076.668768] Hardware name: To be filled by O.E.M. To be filled by O.E.M./To be filled by O.E.M., BIOS 4.6.5 03/08/2016
[ 1076.700801] task: ffffffff819c5460 ti: ffffffff819b0000 task.ti: ffffffff819b0000
[ 1076.723216] RIP: 0010:[<ffffffff81096fd2>]  [<ffffffff81096fd2>] cascade+0xb2/0xc0
[ 1076.745940] RSP: 0018:ffff880a57c03e20  EFLAGS: 00010006
[ 1076.761850] RAX: ffff88016f414000 RBX: 0000000000000000 RCX: ffffffff81e3ed28
[ 1076.783224] RDX: 0000000000000014 RSI: ffff88102ee1fe58 RDI: ffffffff81e3d100
[ 1076.804595] RBP: ffff880a57c03e58 R08: ffff8810a4b1beb8 R09: ffff880a57c03d98
[ 1076.825969] R10: 0000000000000002 R11: ffff880a57c03da0 R12: ffffffff81e3d100
[ 1076.847342] R13: ffff880a57c03e20 R14: 0000000000000014 R15: 0000000000000141
[ 1076.868712] FS:  0000000000000000(0000) GS:ffff880a57c00000(0000) knlGS:0000000000000000
[ 1076.892947] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1076.910157] CR2: 00007f36032b1b80 CR3: 00000012e2aa0000 CR4: 00000000000407f0
[ 1076.931529] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[ 1076.952900] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
[ 1076.974274] Stack:
[ 1076.980305]  ffff88102ee1fe58 ffff880fe53cbd50 d9f141b1e5ffe31b ffffffff81e3d100
[ 1077.002663]  0000000000000000 000000fa54f26809 ffffffff819b70c8 ffff880a57c03ed0
[ 1077.025023]  ffffffff81098c25 ffffffff81e3ed28 ffffffff81e3e928 ffffffff81e3e528
[ 1077.047385] Call Trace:
[ 1077.054717]  <IRQ>
[ 1077.060488]
[ 1077.065016]  [<ffffffff81098c25>] run_timer_softirq+0x2b5/0x340
[ 1077.078849]  [<ffffffff81321c30>] ? timerqueue_add+0x60/0xb0
[ 1077.095802]  [<ffffffff8108f63f>] __do_softirq+0xef/0x280
[ 1077.111974]  [<ffffffff8169929c>] call_softirq+0x1c/0x30
[ 1077.127885]  [<ffffffff8102d365>] do_softirq+0x65/0xa0
[ 1077.143278]  [<ffffffff8108f9d5>] irq_exit+0x115/0x120
[ 1077.158669]  [<ffffffff81699f15>] smp_apic_timer_interrupt+0x45/0x60
[ 1077.177700]  [<ffffffff8169845d>] apic_timer_interrupt+0x6d/0x80
//省略无关

```
从提示上看是触发了如下BUG_ON，也就是说挂在base链上的timer的base竟然是其他CPU的base。
之前分析其他问题也遇到过类似的情况，当时是因为timer还没有delete，就被重新init_timer了,导致base被改掉。

```c
  BUG_ON(tbase_get_base(timer->base) != base);
```

通过汇编代码得知timer地址在rsi中，rsi=ffff88102ee1fe58。然后我们可知这是个neigh_timer_handler。这个timer结构在neighbour结构中.

```bash
crash> dis -r cascade+0xb2
//...
0xffffffff81096f7d <cascade+93>:        mov    0x18(%rsi),%rax	//timer->base
0xffffffff81096f81 <cascade+97>:        mov    %rdi,%r12
0xffffffff81096f84 <cascade+100>:       and    $0xfffffffffffffffc,%rax
0xffffffff81096f88 <cascade+104>:       cmp    %rax,%rdi	//比较timer->base和base
0xffffffff81096f8b <cascade+107>:       je     0xffffffff81096fa3 <cascade+131>
0xffffffff81096f8d <cascade+109>:       jmp    0xffffffff81096fd2 <cascade+178>
//...

crash> timer_list ffff88102ee1fe58
struct timer_list {
  entry = {
    next = 0x0, 
    prev = 0xffff880a57c03e20
  }, 
  expires = 0, 
  base = 0xffff88016f414000, 
  function = 0xffffffff8157f8c0 <neigh_timer_handler>, 
  data = 18446612201820257792, 
  slack = -1, 
  start_pid = -1, 
  start_site = 0x0, 
  start_comm = "\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"
}
```

timer的next为0，且expires为0。timer的expires一旦mod或add后，不会再为0了，除非重新inti_timer/setup_timer。

堆栈neighbour代码，只有在neigh_alloc时会 kzalloc申请neighbour内存(timer所有内容为0， expires自然为0)，然后setup_timer,最终调用do_init_timer，设置timer->base为新的base。

也就是说timer还在运行中，但是neighbour内存释放了，然后重新申请出来后，timer被覆写坏了，最终导致问题。

分析半天neighbour相关代码和公司业务模块代码，没有找到问题原因。

因为这块代码属于内核，且我们的内核业务模块经排查并不会损坏这里。所以怀疑跟内核BUG相关。

几经搜索，改变几次关键字后，终于找到了类似问题的mail list。

[https://www.spinics.net/lists/netdev/msg323883.html](https://www.spinics.net/lists/netdev/msg323883.html)

顺藤摸瓜找到了对应的修改commit

```bash
commit 2c51a97f76d20ebf1f50fef908b986cb051fdff9
Author: Julian Anastasov <ja@ssi.bg>
Date:   Tue Jun 16 22:56:39 2015 +0300

    neigh: do not modify unlinked entries
    
    The lockless lookups can return entry that is unlinked.
    Sometimes they get reference before last neigh_cleanup_and_release,
    sometimes they do not need reference. Later, any
    modification attempts may result in the following problems:
    
    1. entry is not destroyed immediately because neigh_update
    can start the timer for dead entry, eg. on change to NUD_REACHABLE
    state. As result, entry lives for some time but is invisible
    and out of control.
    
    2. __neigh_event_send can run in parallel with neigh_destroy
    while refcnt=0 but if timer is started and expired refcnt can
    reach 0 for second time leading to second neigh_destroy and
    possible crash.
    
    Thanks to Eric Dumazet and Ying Xue for their work and analyze
    on the __neigh_event_send change.
    
    Fixes: 767e97e1e0db ("neigh: RCU conversion of struct neighbour")
    Fixes: a263b3093641 ("ipv4: Make neigh lookups directly in output packet path.")
    Fixes: 6fd6ce2056de ("ipv6: Do not depend on rt->n in ip6_finish_output2().")
    Cc: Eric Dumazet <eric.dumazet@gmail.com>
    Cc: Ying Xue <ying.xue@windriver.com>
    Signed-off-by: Julian Anastasov <ja@ssi.bg>
    Acked-by: Eric Dumazet <edumazet@google.com>
    Signed-off-by: David S. Miller <davem@davemloft.net>

```

根据问题点，反推出环境问题的出错原因：

1. neighbour在释放前被__ipv4_neigh_lookup_noref 得到，但是没有增加ref。
2. dst_neigh_output-->neigh_resolve_output-->neigh_event_send-->__neigh_event_send-->neigh_add_timer-->mod_timer 将neighbour的timer添加到base的对应链表中，并等待运行。
3. 该neighbour被释放
4. 该neighbour被重新alloc，timer部分被set为0，然后setup_timer重新设置了新的base。
5. timer处理时触发BUG_ON。

上面中该问题的解决方式是在相关neighbour函数中增加了dead判断，
也就是说如果neighbour已经是dead了，就不再做进一步处理。避免了问题的发生。

```bash
diff --git a/net/core/neighbour.c b/net/core/neighbour.c
index 3de6542..2237c1b 100644
--- a/net/core/neighbour.c
+++ b/net/core/neighbour.c
@@ -957,6 +957,8 @@ int __neigh_event_send(struct neighbour *neigh, struct sk_buff *skb)
        rc = 0;
        if (neigh->nud_state & (NUD_CONNECTED | NUD_DELAY | NUD_PROBE))
                goto out_unlock_bh;
+       if (neigh->dead)
+               goto out_dead;
 
        if (!(neigh->nud_state & (NUD_STALE | NUD_INCOMPLETE))) {
                if (NEIGH_VAR(neigh->parms, MCAST_PROBES) +
@@ -1013,6 +1015,13 @@ int __neigh_event_send(struct neighbour *neigh, struct sk_buff *skb)
                write_unlock(&neigh->lock);
        local_bh_enable();
        return rc;
+
+out_dead:
+       if (neigh->nud_state & NUD_STALE)
+               goto out_unlock_bh;
+       write_unlock_bh(&neigh->lock);
+       kfree_skb(skb);
+       return 1;
 }
 EXPORT_SYMBOL(__neigh_event_send);
 
@@ -1076,6 +1085,8 @@ int neigh_update(struct neighbour *neigh, const u8 *lladdr, u8 new,
        if (!(flags & NEIGH_UPDATE_F_ADMIN) &&
            (old & (NUD_NOARP | NUD_PERMANENT)))
                goto out;
+       if (neigh->dead)
+               goto out;
 
        if (!(new & NUD_VALID)) {
                neigh_del_timer(neigh);
@@ -1225,6 +1236,8 @@ int neigh_update(struct neighbour *neigh, const u8 *lladdr, u8 new,
  */
 void __neigh_set_probe_once(struct neighbour *neigh)
 {
+       if (neigh->dead)
+               return;
        neigh->updated = jiffies;
        if (!(neigh->nud_state & NUD_FAILED))
                return;
```

后记：
所以有时候快速查找linux bug相关邮件和commit也能提高效率。避免重复劳动，耗费很多精力去定位，最后发现是已知的内核BUG。

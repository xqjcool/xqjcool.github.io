---
title: "[Crash Analysis] A dst_entry double-free issue that gave me a headache for a long time."
date: 2020-09-04
---

公司产品在升级后出现了crash，分析发现在释放skb时，skb->sp = ffff000000000011，不为NULL，在 secpath_put(skb->sp)时，地址访问异常crash。

```bash
PID: 8233   TASK: ffff8800d0503ec0  CPU: 0   COMMAND: "in:imklog"
 #0 [ffff88011fc03710] machine_kexec at ffffffff81059bab
 #1 [ffff88011fc03770] __crash_kexec at ffffffff81105812
 #2 [ffff88011fc03840] crash_kexec at ffffffff81105900
 #3 [ffff88011fc03858] oops_end at ffffffff8168fdc8
 #4 [ffff88011fc03880] die at ffffffff8102e93b
 #5 [ffff88011fc038b0] do_general_protection at ffffffff8168f6be
 #6 [ffff88011fc038e0] general_protection at ffffffff8168ef68
    [exception RIP: skb_release_head_state+49]
    RIP: ffffffff8155e401  RSP: ffff88011fc03990  RFLAGS: 00010286
    RAX: 0000000000000034  RBX: ffff880117870400  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: ffff88011fc0f838  RDI: ffff000000000011
    RBP: ffff88011fc03998   R8: 0000000000000086   R9: 00000000000035ef
    R10: 00000000000003ff  R11: 0000000000000002  R12: ffff880117870400
    R13: ffff8800d306a06a  R14: ffff8800d306a072  R15: 0000000000000010
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88011fc039a0] skb_release_all at ffffffff8155ead2
 #8 [ffff88011fc039b8] consume_skb at ffffffff8155efdc
 #9 [ffff88011fc039d8] __dev_kfree_skb_any at ffffffff8156ee5d
```

进一步分析发现要释放的skb指针和dst_entry共享一块内存（这两个都是kmem_cache_alloc创建的，因为size都是256，共享kmalloc-256这个slab）。即ffff880117870400指向的内存同时被skb指针和dst_entry在使用。

```bash
//作为skb解析
struct sk_buff {
  next = 0x0, 
  prev = 0xffffffff8157ba50 <dst_destroy_rcu>,   //说明被dst_entry使用
  {
    tstamp = {
      tv64 = 0
    }, 
    skb_mstamp = {
      {
        v64 = 0, 
        {
          stamp_us = 0, 
          stamp_jiffies = 0
        }
      }
    }
  }, 
  sk = 0xffff880036882000,                     
  dev = 0xffffffff81aa5e40 <ipv4_dst_ops>,    //dst_entry设置
  cb = "\241rs\201\377\377\377\377\000\000\000\000\000\000\000\000\000\004\207\027\001\210\377\377\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\027[\201\377\377\377\377", 
  _skb_refdst = 0, 
  sp = 0xffff000000000011,       //导致crash原因，sp为异常地址
  len = 0, 
  data_len = 0, 
  mac_len = 14, 
  hdr_len = 0, 
  {
//...
}

//作为dst_entry解析
crash> rtable ffff880117870400
struct rtable {
  dst = {
    callback_head = {
      next = 0x0, 
      func = 0xffffffff8157ba50 <dst_destroy_rcu>
    }, 
    child = 0x0, 
    dev = 0xffff880036882000, 
    ops = 0xffffffff81aa5e40 <ipv4_dst_ops>, 
    _metrics = 18446744071586411169, 
    expires = 0, 
    path = 0xffff880117870400, 
    from = 0x0, 
    xfrm = 0x0, 
    input = 0xffffffff815b1700 <ip_local_deliver>, 
    {
      output = 0x0, 
      __UNIQUE_ID_rh_kabi_hide60 = {
        output = 0x0
      }, 
      {<No data fields>}
    }, 
    flags = 17, 
    pending_confirm = 0, 
    error = 0, 
    obsolete = -1, 
    header_len = 0, 
    trailer_len = 0, 
    tclassid = 0, 
    __pad_to_align_refcnt = {14, 2251816993554432}, 
    __refcnt = {
      counter = 0
    }, 
    __use = 0, 
    lastuse = 4336222445, 
    {
      next = 0x0, 
      rt_next = 0x0, 
      rt6_next = 0x0, 
      dn_next = 0x0
    }, 
    lwtstate = 0x0, 
    rh_reserved3 = 0, 
    rh_reserved4 = 0
  }, 
  rt_genid = 36, 
  rt_flags = 2415919104, 
  rt_type = 3, 
  rt_is_input = 0 '\000', 
  rt_uses_gateway = 0 '\000', 
  rt_iif = 2, 
  rt_gateway = 0, 
  rt_pmtu = 0, 
  rt_uncached = {
    next = 0xffff880036a108c0, 
    prev = 0xffff88011974b4c0
  }
}
```

从上面可以看出来要释放的skb，前面的内存已经被改写成dst_entry的内容了。查看log发现都有crash之前有以下log和warning信息。

```bash
[41554.970852] <Info>[LWCONN]link {0:56333} peer info changed, 113.70.163.73:37452 => 47.55.215.53:37452
[41554.971734] <Info>[LWCONN]link {0:56333} peer info changed, 47.155.215.53:37452 => 113.70.163.73:37452
[41554.976011] <Error>[LWCONN]link {0:56333} there is same 4-tuple link in hash table
[41554.979056] ------------[ cut here ]------------
[41554.981627] WARNING: at lib/list_debug.c:53 __list_del_entry+0x63/0xd0()
[41554.984533] list_del corruption, ffff8801178704c0->next is LIST_POISON1 (dead000000000100)
//...省略
[41555.005100] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G           OE  ------------   3.10.0-514.21.1.el7.x86_64 #1
[41555.008623] Hardware name: Bochs Bochs, BIOS Bochs 01/01/2011
[41555.011465]  ffff88011fc03da0 42e04ad0f25cafa1 ffff88011fc03d58 ffffffff81686f13
[41555.014553]  ffff88011fc03d90 ffffffff81085cb0 ffff8801178704c0 0000000000000000
[41555.017448]  ffffffff81aa5e40 0000000000000000 ffff88011fc101c0 ffff88011fc03df8
[41555.020309] Call Trace:
[41555.022404]  <IRQ>  [<ffffffff81686f13>] dump_stack+0x19/0x1b
[41555.024928]  [<ffffffff81085cb0>] warn_slowpath_common+0x70/0xb0
[41555.027447]  [<ffffffff81085d4c>] warn_slowpath_fmt+0x5c/0x80
[41555.030045]  [<ffffffff816804bb>] ? printk+0x5e/0x75
[41555.032283]  [<ffffffff81333803>] __list_del_entry+0x63/0xd0
[41555.034632]  [<ffffffff8133387d>] list_del+0xd/0x30
[41555.036855]  [<ffffffff815abece>] ipv4_dst_destroy+0x2e/0x40
[41555.039111]  [<ffffffff8157b758>] dst_destroy+0x78/0x120
[41555.041454]  [<ffffffff8157ba5e>] dst_destroy_rcu+0xe/0x20
[41555.043845]  [<ffffffff81139c3d>] rcu_process_callbacks+0x1dd/0x550
[41555.046327]  [<ffffffff8108f63f>] __do_softirq+0xef/0x280
//...省略
```

此后该设备又crash过几次，对比发现都有以下log和warning信息。

从warning栈上看是rcu_process_callbacks调用dst_destroy_rcu释放dst_entry时，发现ffff8801178704c0->next is LIST_POISON1。也就是说该dst_entry在之前已经被释放过(只有释放后会将next置为LIST_POISON1)。

这是导致后面各种crash的根本原因。

那么是什么情况导致 dst_entry重复释放呢，理论上如果是 dst_release的重复调用，会有dst_entry的__refcnt为负的warning。

唯一可能就是在错误逻辑下，rcu_process_callbacks中重复执行了dst_destroy_rcu。

[41554.970852] <Info>[LWCONN]link {0:56333} peer info changed, 113.70.163.73:37452 => 47.55.215.53:37452
[41554.971734] <Info>[LWCONN]link {0:56333} peer info changed, 47.155.215.53:37452 => 113.70.163.73:37452
这两条log提醒了我，这是更新对端ip时打印的，它会调用call_rcu将link 节点挂在当前cpu的rcu_data的nxtlist上，然后在静默期后执行，将link添加到一个hash链表上。

而这两条几乎同时发生的log表明该link在短时间内执行了两次call_rcu（这个不是rcu的标准用法，正常情况下都用call_rcu来清理或释放内存）。

但是这个只会影响link，怎么会导致dst_entry重复释放呢？当时卡在这里好久。

有天盯着那两条log，思考到会不会这两条log是不同的cpu打印的呢？突然有种醒悟的感觉。

顺着这个思路，问题场景慢慢清晰起来。

1. CPUA上收到 linkX控制报文 update peer info，将link挂在 CPUA的 rcu callback上面
2. CPUB上收到 linkX控制报文 update peer info，将link挂在 CPUB的 rcu callback上面。
3. 此时 CPUA和CPUB的 rcu callback链上都有 linkx，造成链表交汇
4. 此后 CPUA和CPUB都会向rcu callback链上添加节点，比如dst_entry节点。必然会有一个rcu callback的链被切断，造成节点未处理。
5. CPUA静默期过后，执行 rcu callback 会执行linkx， dst_entry节点的function， 释放dst_entry;CPUB静默期后也会执行linkx，dst_entry节点的function，释放dst_entry，导致重复释放，messages里出现__list_del warning。
6. 在步骤5期间重复释放的dst_entry可能会被重新申请到，同时为sk_buff使用（sk_buff和rtable都使用kmalloc-256这个slab）和作为dst_entry使用，就会造成异常。

问题原因找到后，心情瞬间舒畅起来。这几天在这个问题上费了不少时间，甚至下班的时间都在思考它。短暂的快乐过后，又要处理下一个棘手的问题了。。。


PS：码字不易，还请小小一赞，感谢感谢！





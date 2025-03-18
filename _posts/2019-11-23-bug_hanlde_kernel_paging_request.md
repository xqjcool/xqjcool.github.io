---
title: "[crash analysis]BUG: unable to handle kernel paging request at ffffeb04000ffb40"
date: 2019-11-23
---
用crash工具打开dordump文件，分析异常栈。

```c

      KERNEL: /usr/lib/debug/lib/modules/3.10.0-514.26.2.el7.x86_64/vmlinux
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 4
        DATE: Fri Nov 22 06:53:47 2019
      UPTIME: 00:10:51
LOAD AVERAGE: 0.09, 0.06, 0.05
       TASKS: 142
    NODENAME: BSE
     RELEASE: 3.10.0-514.26.2.el7.x86_64
     VERSION: #1 SMP Tue Jul 4 15:04:05 UTC 2017
     MACHINE: x86_64  (1996 Mhz)
      MEMORY: 1.9 GB
       PANIC: "BUG: unable to handle kernel paging request at ffffeb04000ffb40"
crash> bt
PID: 2216   TASK: ffff880073f39f60  CPU: 2   COMMAND: "CommuSend"
 #0 [ffff8800517bb5b8] machine_kexec at ffffffff81059beb
 #1 [ffff8800517bb618] __crash_kexec at ffffffff81105822
 #2 [ffff8800517bb6e8] crash_kexec at ffffffff81105910
 #3 [ffff8800517bb700] oops_end at ffffffff81690008
 #4 [ffff8800517bb728] no_context at ffffffff8167fc96
 #5 [ffff8800517bb778] __bad_area_nosemaphore at ffffffff8167fd2c
 #6 [ffff8800517bb7c0] bad_area_nosemaphore at ffffffff8167fe96
 #7 [ffff8800517bb7d0] __do_page_fault at ffffffff81692e4e
 #8 [ffff8800517bb830] do_page_fault at ffffffff81692ff5
 #9 [ffff8800517bb860] page_fault at ffffffff8168f208
    [exception RIP: kmem_cache_free+101]
    RIP: ffffffff811dcd85  RSP: ffff8800517bb910  RFLAGS: 00010282
    RAX: ffffeb04000ffb40  RBX: ffffc90003fed0c8  RCX: 0000000000000006
    RDX: ffffea0000000000  RSI: ffffc90003fed0c8  RDI: ffff880071f28700
    RBP: ffff8800517bb928   R8: 0000000000000092   R9: ffffffffa08374c9
    R10: 0000000000000000  R11: ffff8800517bb636  R12: ffff880071f28700
    R13: 0000000000000005  R14: 0000000000000074  R15: ffff8800517bb9fc
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff8800517bb930] PoolFree at ffffffffa08374c9 [module]
#11 [ffff8800517bb960] _DpFree at ffffffffa07c3cf3 [module]
#12 [ffff8800517bb978] _DpProcess at ffffffffa07c3e82 [module]
#13 [ffff8800517bb9e0] DProcess at ffffffffa084ce0a [module]
#14 [ffff8800517bb9f0] TxHook at ffffffffa0858b07 [module]
#15 [ffff8800517bba30] nf_iterate at ffffffff815a75c0
#16 [ffff8800517bba70] nf_hook_slow at ffffffff815a76a8
#17 [ffff8800517bbaa8] ip_output at ffffffff815b78ce
#18 [ffff8800517bbb08] ip_local_out_sk at ffffffff815b5531
#19 [ffff8800517bbb28] ip_queue_xmit at ffffffff815b58a3
#20 [ffff8800517bbb60] tcp_transmit_skb at ffffffff815cf04f
#21 [ffff8800517bbbd0] tcp_write_xmit at ffffffff815cf68a
#22 [ffff8800517bbc38] __tcp_push_pending_frames at ffffffff815d048e
#23 [ffff8800517bbc50] tcp_push at ffffffff815bed2c
#24 [ffff8800517bbc60] tcp_sendmsg at ffffffff815c25b8
#25 [ffff8800517bbd28] inet_sendmsg at ffffffff815ed854
#26 [ffff8800517bbd58] sock_aio_write at ffffffff81555227
#27 [ffff8800517bbe20] do_sync_write at ffffffff811fe18d
#28 [ffff8800517bbef8] vfs_write at ffffffff811feaf5
#29 [ffff8800517bbf38] sys_write at ffffffff811ff51f
#30 [ffff8800517bbf80] system_call_fastpath at ffffffff81697809
    RIP: 00007f376767579d  RSP: 00007f37575e4c68  RFLAGS: 00000202
    RAX: 0000000000000001  RBX: ffffffff81697809  RCX: 0000000000000018
    RDX: 0000000000000040  RSI: 00007f374720114c  RDI: 0000000000000016
    RBP: 0000000000000000   R8: 0000000000000404   R9: 0000000000000028
    R10: 0000000000000032  R11: 0000000000000293  R12: 00007f374720114c
    R13: 00007f3747201138  R14: 0000000000000001  R15: 0000000000000040
    ORIG_RAX: 0000000000000001  CS: 0033  SS: 002b
```

功能模块加载后，很快就出现crash栈。

从crash位置看 kmem_cache_free+101 ，这是要释放的A结构内存所对应的page地址ffffeb04000ffb40异常，通过page访问page->flags时，page_fault最终crash。

从dump 寄存器配合反汇编可以得知 释放的内存地址为 RBX: ffffc90003fed0c8，这个地址不像正常地址。因为正常内存地址在 [ffff880000000000~FFFF880080000000)之间。

继续查看调用栈，发现是在释放一个模块结构时导致的。查看该结构对应的pool是RDI: ffff880071f28700

crash> struct kmem_cache 0xffff880071f28700
struct kmem_cache {
    size = 320,
    object_size = 312,
    name = 0xffff8800362a76f0 "i915_gem_vma",
...
}

内核会将size大小接近，可共用的cache进行合并。我们模块结构的申请释放就和"i915_gem_vma"共用一个kmem_cache。

继续分析调用栈发现在 DpProcess准备释放时传入的就是这个地址。排除了调用中间被修改的可能。

转换思路，我们看下log中是否有可用信息。

发现在crash前有个 warning信息：

```c

[  654.306627] ------------[ cut here ]------------
[  654.306639] WARNING: at lib/list_debug.c:29 __list_add+0x65/0xc0()
[  654.306644] list_add corruption. next->prev should be prev (ffffc90003fed0c8), but was 0000010c4a41434b. (next=ffff8800363feb38).
[  654.306648] Modules linked in: module(OE) bridge stp llc openvswitch nf_conntrack_ipv6 nf_nat_ipv6 nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat_ipv4 nf_defrag_ipv6 nf_nat it87 hwmon_vid arc4 rt2800usb intel_powerclamp rt2x00usb coretemp intel_rapl rt2800lib crc_ccitt kvm_intel snd_intel_sst_acpi rt2x00lib snd_intel_sst_core snd_soc_sst_mfld_platform kvm mac80211 snd_soc_sst_match snd_soc_core irqbypass snd_compress crc32_pclmul cfg80211 ghash_clmulni_intel cryptd snd_pcm rfkill r8169 sg snd_timer pcspkr mii snd soundcore i2c_i801 shpchp iosf_mbi ip_tables xfs libcrc32c sd_mod crc_t10dif crct10dif_generic i915 i2c_algo_bit drm_kms_helper syscopyarea crct10dif_pclmul sysfillrect crct10dif_common sysimgblt fb_sys_fops crc32c_intel drm serio_raw ahci libahci libata i2c_core video fjes dm_mirror dm_region_hash
[  654.306753]  dm_log dm_mod nf_conntrack [last unloaded: lwconn]
[  654.306764] CPU: 2 PID: 2069 Comm: config_thread/0 Tainted: G           OE  ------------   3.10.0-514.26.2.el7.x86_64 #1
[  654.306768] Hardware name: YanRay Technology B1904/B1904, BIOS 5.6.5 10/19/2017
[  654.306772]  ffff8800738439e8 000000000442e09a ffff8800738439a0 ffffffff81687133
[  654.306780]  ffff8800738439d8 ffffffff81085cb0 ffff8800363febb0 ffff8800363feb38
[  654.306787]  ffffc90003fed0c8 0000000000000002 0000000000000761 ffff880073843a40
[  654.306794] Call Trace:
[  654.306805]  [<ffffffff81687133>] dump_stack+0x19/0x1b
[  654.306814]  [<ffffffff81085cb0>] warn_slowpath_common+0x70/0xb0
[  654.306820]  [<ffffffff81085d4c>] warn_slowpath_fmt+0x5c/0x80
[  654.306827]  [<ffffffff81333955>] __list_add+0x65/0xc0
[  654.306866]  [<ffffffffa07beeb9>] Update+0x1b9/0x2a0 [module]
[  654.306899]  [<ffffffffa07bc97d>] XXXUpdate+0x2d/0x40 [module]
[  654.306935]  [<ffffffffa07e1dd5>] XXXAddNew+0x755/0x980 [module]
[  654.306943]  [<ffffffff8168ea3b>] ? _raw_spin_unlock_bh+0x1b/0x40
[  654.306980]  [<ffffffffa07e20a8>] _XXXAdd+0xa8/0x160 [module]
[  654.307016]  [<ffffffffa07e3060>] XXXAdd+0xd0/0x290 [module]
[  654.307046]  [<ffffffffa07b7b39>] XXXConfigAdd+0x69/0x130 [module]
[  654.307101]  [<ffffffffa07b87c2>] XXXConfigFunc+0x82/0x1330 [module]
[  654.307112]  [<ffffffff8155637a>] ? kernel_recvmsg+0x3a/0x50
[  654.307155]  [<ffffffffa083336a>] ? _XXXRecv+0x4a/0x70 [module]
[  654.307198]  [<ffffffffa08355f6>] ? XXXRecv+0xb6/0x190 [module]
[  654.307239]  [<ffffffffa0835f05>] _XXXBBBThread+0x35/0x110 [module]
[  654.307303]  [<ffffffffa0835ed0>] ? _XXXAAAThread+0x110/0x110 [module]
[  654.307335]  [<ffffffff810b0a4f>] kthread+0xcf/0xe0
[  654.307362]  [<ffffffff810b0980>] ? kthread_create_on_node+0x140/0x140
[  654.307382]  [<ffffffff81697758>] ret_from_fork+0x58/0x90
[  654.307390]  [<ffffffff810b0980>] ? kthread_create_on_node+0x140/0x140
[  654.307395] ---[ end trace 4b0d7a0f27c571a5 ]---
```

WARNING信息中prev (ffffc90003fed0c8) 这个地址正好是我们crash时释放的地址。深挖这个warning，这是另一个子模块申请B结构后，并将B结构加入到全局链表操作。查看全局链表结构

crash> p &g_XXX[0]->xxxTbl          
$3 = (XXXTBL *) 0xffffc90003fed0b0
crash> XXXTBL 0xffffc90003fed0b0 -ox
typedef struct _XXXTBL {
  [ffffc90003fed0b0] uint32_t Cnt;
  [ffffc90003fed0b8] struct _XXXBLOCK **Tbl;
  [ffffc90003fed0c0] YYYTBL *yyyTbl;
  [ffffc90003fed0c8] list_head List;
  [ffffc90003fed0d8] timer_list Timer;
  [ffffc90003fed150] rwlock_t Lock;
} XXXTBL;
SIZE: 0xa8

 异常地址ffffc90003fed0c8是这个全局结构的List的地址ffffc90003fed0c8，也就是说该地址在某个时刻写到了cache 内存里，并被当作地址使用了。查看这个List确实被改写了。

crash> XXXTBL.List 0xffffc90003fed0b0 
  List = {
    next = 0x1384a41434b, 
    prev = 0x6808010a00000600
  }

 next=ffff8800363feb38 ，经查看该地址属于 kmem_cache "i915_gem_vma"。这不正确。 这个XXXBLOCK结构大于"i915_gem_vma" cache的size。不应该使用同一个cache。

我们模块中存放了不同cache对应的idx索引。经查看B结构 XXXBLOCK的索引和 A结构的索引是相同的。

于是查看代码修改，发现有一处字符串比较，本应是 相同则进入分支处理，结果使用了

if （strncmp()）{do_sth},该函数是字符串不同，返回非零值，进入了分支处理。导致了异常发生。修复后验证OK。

PS：strcmp/strncmp的条件一定要加上 0 == strcmp 或者0 != strcmp， 以来方便理解代码期望，而来不容易发错。此处错误容易忽视。

=====================================继续分析的分割线=======================================

虽然问题解决了，但是这个异常地址被kmem_cache alloc出来，导致最终释放，还是没有想明白，于是继续分析。

后来想明白了，XXXBLOCK和AAA用相同的cache，XXXBLOCK申请出来的内存cache是320bytes，但是XXXBLOCK是328个bytes，XXXBLOCK的最后8个字节是 list的prev。

第一次添加到 XXXTBL.list的时候， prev会被赋予 XXXTBL.list地址，所以 这8个字节的值为 ffffc90003fed0c8。

如果对cache内存管理熟悉的话会知道，内存object头8个字节是，是该object下一个可用 object的地址。

简单示意图如下。

1. 刚开始：

memory=可用内存的头部

|<---memory

|--地址指向2--|---可使用内存1----||---地址指向3---|----可使用内存2----||---地址指向4---|-------可使用内存3-------|

2. 申请XXXBLOCK，读取freelist得到内存地址，这时是指向可用内存1的头部。同时将头部8个字节赋予freelist，以备下一次使用，freelist此时指向可用内存2的头部地址。

配置XXXBLOCK，并将XXXBLOCK的list加到XXXTbl的list上，因为写越界，

原本指向可用内存3个地址，被写成了ffffc90003fed0c8。

 memory------------------------>|

|------------XXXBLOCK---------|ffffc90003fed0c8|-------可使用内存2-------||---地址指向4---|-----可使用内存3-------|

3. AAA申请时，读取memory得到内存地址，这时是指向可用内存2的头部。同时将头部8个字节赋予memory，以备下一次使用，mem
                  memory=ffffc90003fed0c8 指向了全局XXXTbl的List地址。

|-------------XXXBLOCK---------|----------------A结构--------------------------||---地址指向4---|-------可使用内存3-------|

4. 再次申请AAA时，读取memory得到内存地址ffffc90003fed0c8，配置该处内存(将XXXTbl的内容覆写)。

5. 释放第二次申请的A结构时，因为地址异常，最终crash。

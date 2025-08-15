---
title: "A system crash caused by an nf_conn use-after-free"
date: 2025-08-15
---

# nf_conn use-after-free 导致的系统crash

## 1. 问题现象

QA那边上报了一个系统crash问题，据说复现概率挺高，每次都crash在同样位置。
于是查看crash log，发现是expectaion的timer超市处理中， nf_ct_unlink_expect_report 执行时，发生空地址引用导致crash。

## 2. 初步分析

日志内容如下

```bash
[ 1907.969774] ------------[ cut here ]------------
[ 1907.969776] WARNING: CPU: 5 PID: 0 at net/netfilter/nf_conntrack_expect.c:55 nf_ct_unlink_expect_report+0x6d/0x1f0
[ 1907.969792] CPU: 5 PID: 0 Comm: swapper/5 Kdump: loaded Tainted: G           O       6.1 #1
[ 1907.969794] Hardware name: Supermicro SYS-2049P-TN8R-FI005/X11QPL, BIOS 3.3 02/19/2020
[ 1907.969795] RIP: 0010:nf_ct_unlink_expect_report+0x6d/0x1f0
[ 1907.969798] Code: b0 00 00 00 48 85 c0 75 25 eb 1f 44 8b 0d 13 57 fb 00 45 8b 50 1c 4c 01 c0 45 85 d2 74 df 45 39 ca 74 da 48 8b 99 b0 00 00 00 <0f> 0b 31 c0 48 83 bf 80 00 00 00 00 0f 85 13 01 00 00 48 8b 4f 10
[ 1907.969800] RSP: 0018:ffffc90008940e30 EFLAGS: 00010246
[ 1907.969802] RAX: 0000000000000000 RBX: ffffffff81e22440 RCX: ffff8882668de800
[ 1907.969803] RDX: 0000000000000000 RSI: 0000000000000000 RDI: ffff888266150000
[ 1907.969804] RBP: ffffc90008940e58 R08: ffff88c84f792980 R09: 00000000ffffffc0
[ 1907.969806] R10: ffff8897e0c9cc08 R11: ffffffff80ec9470 R12: ffffffff80ec9470
[ 1907.969806] R13: 0000000000000000 R14: ffff888266150000 R15: ffff888266150078
[ 1907.969807] FS:  0000000000000000(0000) GS:ffff8897e0c80000(0000) knlGS:0000000000000000
[ 1907.969809] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1907.969810] CR2: 00007ff70fb51e4c CR3: 0000000001c14003 CR4: 00000000007706e0
[ 1907.969812] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[ 1907.969813] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[ 1907.969814] PKRU: 55555554
[ 1907.969814] Call Trace:
[ 1907.969815]  <IRQ>
[ 1907.969817]  ? __warn+0x197/0x270
[ 1907.969819]  ? nf_ct_unlink_expect_report+0x6d/0x1f0
[ 1907.969821]  ? report_bug+0x1af/0x240
[ 1907.969825]  ? nf_ct_unlink_expect_report+0x6d/0x1f0
[ 1907.969827]  ? handle_bug+0x41/0x70
[ 1907.969830]  ? exc_invalid_op+0x1b/0x50
[ 1907.969832]  ? asm_exc_invalid_op+0x1b/0x20
[ 1907.969834]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1907.969837]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1907.969839]  ? nf_ct_unlink_expect_report+0x6d/0x1f0
[ 1907.969840]  nf_ct_expectation_timed_out+0x2b/0x90
[ 1907.969843]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1907.969844]  call_timer_fn+0x2f/0x110
[ 1907.969848]  run_timer_softirq+0x616/0x700
[ 1907.969850]  ? tick_sched_timer+0x129/0x290
[ 1907.969853]  __do_softirq+0xdc/0x2ab
[ 1907.969855]  irq_exit_rcu+0x6c/0xa0
[ 1907.969858]  sysvec_apic_timer_interrupt+0x76/0x90
[ 1907.969861]  </IRQ>
//省略无关
[ 1907.969890] BUG: kernel NULL pointer dereference, address: 0000000000000010
[ 1908.053228] #PF: supervisor write access in kernel mode
[ 1908.115727] #PF: error_code(0x0002) - not-present page
[ 1908.177195] PGD 0 P4D 0 
[ 1908.207457] Oops: 0002 [#1] SMP NOPTI
[ 1908.251242] CPU: 5 PID: 0 Comm: swapper/5 Kdump: loaded Tainted: G        W  O       6.1 #1
[ 1908.351180] Hardware name: Supermicro SYS-2049P-TN8R-FI005/X11QPL, BIOS 3.3 02/19/2020
[ 1908.445924] RIP: 0010:nf_ct_unlink_expect_report+0xd4/0x1f0
[ 1908.512589] Code: 8b 08 0d 00 00 4f 8b 04 c1 41 ff 48 04 4c 8b 07 4c 8b 4f 08 4d 89 01 4d 85 c0 74 04 4d 89 48 08 48 89 4f 08 8b 8f a8 00 00 00 <fe> 4c 08 10 48 8b 47 70 48 8b 80 b0 00 00 00 48 8b 80 90 0c 00 00
[ 1908.737335] RSP: 0018:ffffc90008940e30 EFLAGS: 00010246
[ 1908.799836] RAX: 0000000000000000 RBX: ffffffff81e22440 RCX: 0000000000000000
[ 1908.885220] RDX: 0000000000000000 RSI: 0000000000000000 RDI: ffff888266150000
[ 1908.970599] RBP: ffffc90008940e58 R08: 0000000000000000 R09: ffff88c84f7929a8
[ 1909.055984] R10: ffff8897e0c9cc08 R11: ffffffff80ec9470 R12: ffffffff80ec9470
[ 1909.141370] R13: 0000000000000000 R14: ffff888266150000 R15: ffff888266150078
[ 1909.226754] FS:  0000000000000000(0000) GS:ffff8897e0c80000(0000) knlGS:0000000000000000
[ 1909.323573] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1909.392322] CR2: 0000000000000010 CR3: 0000000001c14003 CR4: 00000000007706e0
[ 1909.477704] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[ 1909.563084] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[ 1909.648468] PKRU: 55555554
[ 1909.680815] Call Trace:
[ 1909.710035]  <IRQ>
[ 1909.734062]  ? __die_body+0x82/0x130
[ 1909.776804]  ? page_fault_oops+0x419/0x510
[ 1909.825789]  ? do_user_addr_fault+0x4d6/0x6a0
[ 1909.877890]  ? report_bug+0x1af/0x240
[ 1909.921676]  ? ct_nmi_enter+0x94/0xc0
[ 1909.965457]  ? exc_page_fault+0x4f/0xa0
[ 1910.011321]  ? asm_exc_page_fault+0x27/0x30
[ 1910.061349]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1910.116568]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1910.171793]  ? nf_ct_unlink_expect_report+0xd4/0x1f0
[ 1910.231178]  nf_ct_expectation_timed_out+0x2b/0x90
[ 1910.288482]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 1910.343703]  call_timer_fn+0x2f/0x110
[ 1910.387488]  run_timer_softirq+0x616/0x700
[ 1910.436469]  ? tick_sched_timer+0x129/0x290
[ 1910.486496]  __do_softirq+0xdc/0x2ab
[ 1910.529241]  irq_exit_rcu+0x6c/0xa0
[ 1910.570944]  sysvec_apic_timer_interrupt+0x76/0x90
[ 1910.628249]  </IRQ>
//省略无关
```

我们发现crash前有个warn， 正好对应代码中的 WARN_ON(!master_help)， 也就是说 master_help指针为NULL。

```c
/* nf_conntrack_expect helper functions */
void nf_ct_unlink_expect_report(struct nf_conntrack_expect *exp,
				u32 portid, int report)
{
	struct nf_conn_help *master_help = nfct_help(exp->master);
	struct net *net = nf_ct_exp_net(exp);
	struct nf_conntrack_net *cnet;

	WARN_ON(!master_help);                  //mmaster_help为NULL， 触发warn
	WARN_ON(timer_pending(&exp->timeout));

	hlist_del_rcu(&exp->hnode);

	cnet = nf_ct_pernet(net);
	cnet->expect_count--;

	hlist_del_rcu(&exp->lnode);
	master_help->expecting[exp->class]--;    //空指针引用

	nf_ct_expect_event_report(IPEXP_DESTROY, exp, portid, report);
	nf_ct_expect_put(exp);

	NF_CT_STAT_INC(net, expect_delete);
}
```

很容易就定位到造成问题的直接原因。但是为什么master_help为NULL呢？从代码逻辑上看这里时不应该为NULL的。

## 3. 分析coredump

使用crash工具打开coredump文件，查看相关变量

```bash
crash> kmem ffff888266150000      //exp
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff8881044a0200      216          1       111      3     8k  nf_conntrack_expect
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0009985400  ffff888266150000     0     37          1    36
  FREE / [ALLOCATED]
  [ffff888266150000]

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0009985400  266150000 ffff8881044a0200        0  1 200000000010200 slab,head
crash> nf_conntrack_expect.master -x ffff888266150000  //master
  master = 0xffff8882668de800,
crash> kmem 0xffff8882668de800
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff888100042a00      512       9533     13216    413    16k  kmalloc-512
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea00099a3700  ffff8882668dc000     0     32         14    18
  FREE / [ALLOCATED]
   ffff8882668de800                                          //master指向的nf_conn内存已被释放!!!

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00099a3780  2668de000 dead000000000400        0  0 200000000000000
crash> nf_conn.ext -x 0xffff8882668de800
  ext = 0xffff88c84f792980,
crash> kmem 0xffff88c84f792980
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff888100042700      128      16907     31872    996     4k  kmalloc-128
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea01213de480  ffff88c84f792000     3     32         17    15
  FREE / [ALLOCATED]
   ffff88c84f792980  (cpu 11 cache)                      //master->ext指向的nf_ct_ext内存已被释放!!!

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea01213de480 484f792000 ffff888100042700 ffff88c84f792b80  1 e00000000000200 slab

crash> nf_ct_ext -x ffff88c84f792980
struct nf_ct_ext {
  offset = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x22, 0x0, 0x8, 0x0, 0x0},  //offset为0
  len = 0x58,
  gen_id = 0x1,
  data = 0xffff88c84f7929a0 ""
}

```

也就是说因为 nf_conn和nf_ct_ext对应内存已经被释放，导致 渠道offset[0]的值是0，master_help = nfct_help(exp->master)函数返回NULL。 
进一步的原因是内存的use-after-free。

## 4. 验证推断

查看代码在 nf_conntrack_free 之前 ，有调用 nf_ct_remove_expectations 会将所有的expactation的timer移除。

于是猜想是不是在移除timer时，timer刚好在执行，导致移除timer失败。这样nf_conn被释放后，timer继续执行出现use-after-free。
且不说这种概率超小，不应该这么频繁出现，而且在这种极端情况下nf_conn和nf_ct_ext内存还没有被改写，就算访问到也是原有数据。

但还是想着验证下吧，于是在nf_conntrack_alloc, nf_conntrack_free 中添加日志，编译debug image，期望能在复现后看到nf_conn已被释放的日志。

很快QA就又复现问题了，查看debug 日志，奇怪的是，根本没有问题nf_conn的释放日志，更奇怪的是也没有问题nf_conn的申请日志！
这就有点诡异了。

## 5. 迭代分析和验证

为了帮助定位，开启了SLUB_DEBUG编译选项，开启后会在slab的后面保存申请和释放的函数信息。

```c
static void *___slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node,
			  unsigned long addr, struct kmem_cache_cpu *c, unsigned int orig_size)
{
//忽略无关
	if (kmem_cache_debug(s)) {
		freelist = alloc_single_from_new_slab(s, slab, orig_size);

		if (unlikely(!freelist))
			goto new_objects;

		if (s->flags & SLAB_STORE_USER)
			set_track(s, freelist, TRACK_ALLOC, addr);    //会将申请函数地址存入

		return freelist;
	}
//忽略无关
}
```

再次复现后，查看问题 nf_conn的slab中的申请释放信息，发现对应内存在crash前曾被用作skb的buffer，没有有效信息可用。

追查代码，定时器是在 nf_ct_expect_insert 是添加的，于是在这里增加调用栈，编译debug image，再去复现。
复现后，发现在crash前，就只有一个 nf_ct_expect_insert ，后续的crash就是这个expectation。

为了查看nf_ct_expect_insert时的nf_conn信息，我在这个函数中调用了panic函数，让它在这里产生coredump。
再次复现后，分析coredump，在nf_conn的slab中找到了申请函数

```bash
crash> kmem ffff888136fcd000    //nf_conn
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff888100048f40      512      14303     15666    746    32k  kmalloc-512
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0004dbf200  ffff888136fc8000     0     21         21     0
  FREE / [ALLOCATED]
  [ffff888136fcce00]
 
      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0004dbf340  136fcd000 dead000000000400        0  0 200000000000000
crash> struct kmem_cache.offset,inuse -x ffff888100048f40
  offset = 0x208,
  inuse = 0x208,
  object_size = 0x200,
crash> p (0xffff888136fcd000 + 0x208 + 0x8) -x                       
$4 = 0xffff888136fcd210            //track

crash> track ffff888136fcd210 -x
struct track {
  addr = 0xffffffff80f13d8d,
  handle = 0x7cf00be,
  cpu = 0xd,
  pid = 0x1247,
  when = 0xfffc8e5c
}
crash> rd -s ffff888136fcd210
ffff888136fcd210:  xt_ct_tg_check_v2+29    //申请函数
```

查看代码发现nf_conn的申请不是 nf_conntrack_alloc， 而是 nf_ct_tmpl_alloc。同样的，释放函数不是 nf_conntrack_free 而是 nf_ct_tmpl_free 。
到这里才了解为什么之前看不到释放日志。是因为没考虑到template的申请和释放。

并且nf_conn template 的释放并不会调用 nf_ct_remove_expectations 来删除对应的timer。这就会导致 nf_conn释放后，timer依然存在，并在超时后调用。最终出现use-after-free问题。

```c
void nf_ct_destroy(struct nf_conntrack *nfct)
{
	struct nf_conn *ct = (struct nf_conn *)nfct;

	pr_debug("%s(%p)\n", __func__, ct);
	WARN_ON(refcount_read(&nfct->use) != 0);

	if (unlikely(nf_ct_is_template(ct))) {
		nf_ct_tmpl_free(ct);                    //释放nf_conn template
		return;                                 //返回
	}

	if (unlikely(nf_ct_protonum(ct) == IPPROTO_GRE))
		destroy_gre_conntrack(ct);

	/* Expectations will have been removed in clean_from_lists,
	 * except TFTP can create an expectation on the first packet,
	 * before connection is in the list, so we need to clean here,
	 * too.
	 */
	nf_ct_remove_expectations(ct);          //删除expectaion的timer

	if (ct->master)
		nf_ct_put(ct->master);

	pr_debug("%s: returning ct=%p to slab\n", __func__, ct);
	nf_conntrack_free(ct);
}
```

重新在 nf_ct_tmpl_alloc 和 nf_ct_tmpl_free 中添加日志，并编译image复现。
复现后，查看日志，确实如此。

```bash
//申请
[ 1978.316487] nf_conntrack: [nf_ct_tmpl_alloc:580] nf_conn:ffff8881391e3800 ext:0

//插入
[ 2131.989389] [nf_ct_expect_insert:417] exp:ffff88823aac8008 master:ffff8881391e3800 ext:ffff888286a3c500 jiffies:4296796140 timeout:300 expires:4297096140

//释放
[ 2140.352889] nf_conntrack: [nf_ct_tmpl_free:594] nf_conn:ffff8881391e3800 ext:6b6b6b6b6b6b6b6b
[ 2140.352891] CPU: 0 PID: 4691 Comm: netd Kdump: loaded Tainted: G        W  O       6.1 #16
[ 2140.352892] Hardware name: Supermicro SYS-2049P-TN8R-FI005/X11QPL, BIOS 3.3 02/19/2020
[ 2140.352893] Call Trace:
[ 2140.352893]  <TASK>
[ 2140.352894]  nf_ct_tmpl_free+0x4f/0x60
[ 2140.352896]  nf_ct_destroy+0xce/0x290
[ 2140.352898]  xt_ct_tg_destroy+0x78/0xc0
[ 2140.352900]  xt_ct_tg_destroy_v1+0x12/0x20
[ 2140.352902]  cleanup_entry+0x115/0x1b0
[ 2140.352904]  __do_replace+0x3ab/0x530
[ 2140.352906]  ? do_ipt_set_ctl+0x5ef/0x6c0
[ 2140.352907]  do_ipt_set_ctl+0x5ef/0x6c0
[ 2140.352909]  nf_setsockopt+0x1a8/0x2e0
[ 2140.352911]  raw_setsockopt+0x7b/0x120
[ 2140.352912]  sock_common_setsockopt+0x18/0x30
[ 2140.352913]  __sys_setsockopt+0xb9/0x130
[ 2140.352915]  __x64_sys_setsockopt+0x21/0x30
[ 2140.352917]  do_syscall_64+0x49/0xa0
[ 2140.352919]  ? irqentry_exit+0x12/0x40
[ 2140.352920]  entry_SYSCALL_64_after_hwframe+0x64/0xce

//crash
[ 2433.066066] general protection fault, probably for non-canonical address 0x6b6b6b6b6b6b6b6b: 0000 [#1] SMP NOPTI
[ 2433.187797] CPU: 10 PID: 66 Comm: ksoftirqd/10 Kdump: loaded Tainted: G        W  O       6.1 #16
[ 2433.293977] Hardware name: Supermicro SYS-2049P-TN8R-FI005/X11QPL, BIOS 3.3 02/19/2020
[ 2433.306651] nf_conntrack: [__nf_conntrack_alloc:1729] nf_conn:ffff8882a9268440 jiffies:4297097457
[ 2433.388722] RIP: 0010:nf_ct_unlink_expect_report+0x2d/0x1f0
[ 2433.388730] Code: 00 00 55 48 89 e5 41 56 53 48 83 ec 18 65 48 8b 04 25 28 00 00 00 48 89 45 e8 48 8b 4f 70 4c 8b 81 e8 00 00 00 4d 85 c0 74 39 <41> 0f b7 00 48 85 c0 74 30 41 83 78 1c 00 75 11 4c 01 c0 48 8b 99
[ 2433.388732] RSP: 0018:ffffc9000ce0fce0 EFLAGS: 00010202
[ 2433.848812] RAX: a79bfdc906a58200 RBX: ffff88823aac8088 RCX: ffff8881391e3800
[ 2433.934200] RDX: 0000000000000000 RSI: 0000000000000000 RDI: ffff88823aac8008
[ 2434.019584] RBP: ffffc9000ce0fd08 R08: 6b6b6b6b6b6b6b6b R09: 0000000000000000
[ 2434.104964] R10: ffff8897e0f1cc00 R11: ffffffff80ee7e00 R12: ffffffff80ee7e00
[ 2434.190349] R13: 0000000000000000 R14: ffff88823aac8008 R15: ffff88823aac8088
[ 2434.275728] FS:  0000000000000000(0000) GS:ffff8897e0f00000(0000) knlGS:0000000000000000
[ 2434.372555] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 2434.441296] CR2: 00007f980bc97000 CR3: 0000000107734003 CR4: 00000000007706e0
[ 2434.526684] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[ 2434.612066] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[ 2434.697449] PKRU: 55555554
[ 2434.729791] Call Trace:
[ 2434.759017]  <TASK>
[ 2434.784079]  ? __die_body+0x82/0x130
[ 2434.826826]  ? die_addr+0xaa/0xe0
[ 2434.866446]  ? exc_general_protection+0x13a/0x1e0
[ 2434.922711]  ? asm_exc_general_protection+0x27/0x30
[ 2434.981054]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 2435.036276]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 2435.091503]  ? nf_ct_unlink_expect_report+0x2d/0x1f0
[ 2435.150885]  nf_ct_expectation_timed_out+0x2b/0x90
[ 2435.208189]  ? nf_ct_expect_dst_hash+0x120/0x120
[ 2435.263415]  call_timer_fn+0x2f/0x110
[ 2435.307195]  run_timer_softirq+0x616/0x700
[ 2435.356179]  ? newidle_balance+0x299/0x320
[ 2435.405166]  __do_softirq+0xdc/0x2ab
[ 2435.447904]  run_ksoftirqd+0x1c/0x30
[ 2435.490649]  smpboot_thread_fn+0xe8/0x1b0
[ 2435.538595]  kthread+0x269/0x2a0
[ 2435.577179]  ? __smpboot_create_thread+0x220/0x220
[ 2435.634479]  ? kthreadd+0x380/0x380
[ 2435.676187]  ret_from_fork+0x1f/0x30
[ 2435.718930]  </TASK>

```

## 6. 问题场景

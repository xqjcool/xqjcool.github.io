---
title: "[Crash Analysis][MIPS] An exception caused by an OpenWrt patch"
date: 2020-04-24
---
公司mips设备在运行中总是会莫名的crash，crash点不确定，有在驱动中，有在socket处理中，等等。

crash其一：
```c
//...
<4>[  158.547434] Process ksoftirqd/0 (pid: 3, threadinfo=8fc62000, task=8fc50b20, tls=00000000)
<4>[  158.555684] Stack : 8e669000 8d45d184 5e8ded5e 00000000 296a99bf 8007b788 8e669120 8fe45880
<4>[  158.555684]         00010001 000003e8 0000000f 00030000 00000000 01000003 00000010 30000000
<4>[  158.555684]         8fc63e00 80089b80 01000000 8b235c60 4b0f8002 0b1fce60 00000000 80470000
<4>[  158.555684]         8120e020 8fc13c78 8fc63df8 8fc63e00 00000080 80466040 0000012c 80470000
<4>[  158.555684]         00000001 802afb34 e0998220 00000080 8e66901c 804c51f0 80470888 ffffc8a5
<4>[  158.555684]         ...
<4>[  158.591848] Call Trace:
<4>[  158.594339] [<801e87d0>] dql_completed+0x10/0x16c
<4>[  158.599099] [<80274f18>] fe_poll+0x184/0x710
<4>[  158.603464] [<802afb34>] net_rx_action+0x130/0x2e0
<4>[  158.608323] [<8002fce8>] __do_softirq+0x294/0x2e0
<4>[  158.613070] [<8002fd6c>] run_ksoftirqd+0x38/0x6c
<4>[  158.617746] [<800488f0>] smpboot_thread_fn+0x18c/0x1bc
<4>[  158.622945] [<80045fd8>] kthread+0xd8/0xec
<4>[  158.627084] [<80005478>] ret_from_kernel_thread+0x14/0x1c
<4>[  158.632490] 
<4>[  158.633996] 
<4>[  158.633996] Code: 8c830024  01433023  00c5102b <00020336> 8c870020  8c8b002c  00c73023  28c20000  00654821 
```
经分析是触发了这个BUG_ON导致的。
寄存器 $10 (num_queued) - $3 （num_completed）= $6 =0x1a6  < $4 (count) = 0x1aa
满足BUG_ON条件，所以crash了。
于是分析驱动是不是有问题导致错误，分析很久没有发现错误地方。

crash其二：
```c
//...
<4>[358894.025935] Process brctl (pid: 23301, threadinfo=83702000, task=87314a10, tls=772c3d48)
<4>[358894.034376] Stack : 00000010 00000000 80410000 80434514 80433870 06000100 000003e8 000005dc
<4>[358894.034376]    00000000 55860c15 83744800 8768bcc0 8768bcc0 83744864 00000000 802989dc
<4>[358894.034376]    00000000 00000000 00000000 00000000 00000000 00000000 80433870 00000000
<4>[358894.034376]    00000001 804b0000 8718d000 8768bcc0 00000000 87c50c00 00000000 00000000
<4>[358894.034376]    772bb518 772bcd8c 00000000 80298ba4 87c50c00 00000100 00000000 00000000
<4>[358894.034376]    ...
<4>[358894.071454] Call Trace:
<4>[358894.074077] [<80287e24>] sk_filter_trim_cap+0x58/0x160
<4>[358894.079488] [<802989dc>] netlink_broadcast_filtered+0x214/0x3c0
<4>[358894.085701] [<80298ba4>] netlink_broadcast+0x1c/0x28
<4>[358894.090928] [<8029ad48>] nlmsg_notify+0x70/0xf8
<4>[358894.095699] [<80286d80>] rtmsg_ifinfo_send+0x24/0x30
<4>[358894.100931] [<80275448>] __dev_notify_flags+0x34/0xac
<4>[358894.106245] [<802755b4>] __dev_set_promiscuity+0xf4/0x114
<4>[358894.111917] [<802758a4>] dev_set_promiscuity+0x24/0x5c
<4>[358894.117330] [<80339c3c>] br_manage_promisc+0x44/0x7c
<4>[358894.122553] [<8033a3e8>] br_add_if+0x2f0/0x4a8
<4>[358894.127244] [<8028aedc>] dev_ifsioc+0x30c/0x330
<4>[358894.132022] [<8028b4ac>] dev_ioctl+0x5ac/0x6d4
<4>[358894.136735] [<8011b0e4>] do_vfs_ioctl+0x4e8/0x5a0
<4>[358894.141692] [<8011b1ec>] SyS_ioctl+0x50/0x98
<4>[358894.146210] [<80062c2c>] syscall_common+0x30/0x54
```
经分析是brctl 操作桥，然后netlink通知，通过sk_filter_trim_cap 调用ebpf。 但是sk->filter 指针异常，导致crash。正常情况这个指针为NULL，这里是异常值。
也没有找到线索。
还有在内核其他位置的crash，总之各种各样，简直无从下手。

不过在crash过程中发现部分crash log，在crash之前有bad page warning信息。

```c
<1>[   81.764968] BUG: Bad page state in process ksoftirqd/0  pfn:03918
<0>[   81.771364] page:81072300 count:-1 mapcount:0 mapping:  (null) index:0x0
<0>[   81.778283] flags: 0x0()
<1>[   81.780926] page dumped because: nonzero _count
//...
<4>[   81.862823] CPU: 0 PID: 3 Comm: ksoftirqd/0 Tainted: G        W       4.4.59 #0
<4>[   81.870387] Stack : 803d2704 00000000 00000001 80430000 87c34c94 8041ad63 803b3314 00000003
<4>[   81.870387]     8048378c 81072300 00000040 87db0000 804a0000 800a7888 803b8ab4 80410000
<4>[   81.870387]     00000003 81072300 803b6e38 87c45b9c 804a0000 800a5804 00000040 87db0000
<4>[   81.870387]     00000000 801f7b00 00000000 00000000 00000000 00000000 00000000 00000000
<4>[   81.870387]     00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
<4>[   81.870387]     ...
<4>[   81.907406] Call Trace:
<4>[   81.909967] [<80071eb0>] show_stack+0x50/0x84
<4>[   81.914484] [<800d65f8>] bad_page+0xec/0x11c
<4>[   81.918908] [<800d8c28>] get_page_from_freelist+0x590/0x810
<4>[   81.924684] [<800d8f80>] __alloc_pages_nodemask+0xd8/0x6f0
<4>[   81.930371] [<800d9700>] __alloc_page_frag+0x4c/0x174
<4>[   81.935602] [<80239f8c>] ag71xx_fill_rx_buf+0x60/0xf0
<4>[   81.940841] [<8023aed0>] ag71xx_poll+0x3d4/0x5c0
<4>[   81.945620] [<80272480>] net_rx_action+0x10c/0x2c8
<4>[   81.950602] [<80084410>] __do_softirq+0x250/0x298
<4>[   81.955465] [<80084480>] run_ksoftirqd+0x28/0x60
<4>[   81.960263] [<8009aed4>] smpboot_thread_fn+0x158/0x188
<4>[   81.965593] [<80098904>] kthread+0xd8/0xec
<4>[   81.969845] [<80060878>] ret_from_kernel_thread+0x14/0x1c
```
开始怀疑这些crash是由于double free导致。

对业务模块代码进行了走查，没有发现会导致double free的地方。为了方便暴露问题，开启内核的 CONFIG_DEBUG_VM 配置项，重新编译内核，制作bin包升级设备。
开启这个选项后，释放page时，如果page count已经为0，则触发BUG，打印调用栈。

开启 CONFIG_DEBUG_VM 后，设备crash栈。
```c
<6>[136515.650812] br0: port 2(eth0.1) entered disabled state
<0>[136515.650973] page:810fbf40 count:0 mapcount:0 mapping:  (null) index:0x0
<0>[136515.650982] flags: 0x0()
<1>[136515.650997] page dumped because: VM_BUG_ON_PAGE(({ union { typeof((&page->_count)->counter) __val; char __c[1]; } __u; if (1) __read_once_size(&((&page->_count)->counter), __u.__c, sizeof((&page->_count)->counter)); else __read_once_size_nocheck(&((&page->_count)->counter), __u.__c, sizeof((&page->_count)->counter)); __u.__val; }) == 0)
<4>[136515.651013] Kernel bug detected[#1]:
<4>[136515.651027] CPU: 1 PID: 11 Comm: ksoftirqd/1 Not tainted 4.4.167 #0
<4>[136515.651035] task: 8fc54850 ti: 8fc80000 task.ti: 8fc80000
<4>[136515.651048] $ 0   : 00000000 00000001 00000000 00010000
<4>[136515.651056] $ 4   : 8047b634
<6>[136515.651057] br0: port 1(eth0.2) entered disabled state
<4>[136515.651069]  00000001 0000dfac 00000000
<4>[136515.651079] $ 8   : 00000002 804df170 00000000 297d203b
<4>[136515.651089] $12   : 00000001 8bff0140 00000000 00000000
<4>[136515.651099] $16   : 83aa9000 81d5a368 00000000 8d4d0000
<4>[136515.651110] $20   : 8d4d0000 00000000 00000014 00000001
<4>[136515.651121] $24   : 00000002 8d41d0a8                  
<4>[136515.651134] $28   : 8fc80000 8fc81ae8 02080020 800a2ec4
<4>[136515.651136] Hi    : 000c9a2c
<4>[136515.651139] Lo    : 43e40ed8
<4>[136515.651192] epc   : 800a2ec4 __free_page_frag+0x4c/0x9c
<4>[136515.651201] ra    : 800a2ec4 __free_page_frag+0x4c/0x9c
<4>[136515.651210] Status: 11007c03	KERNEL EXL IE 
<4>[136515.651214] Cause : 50800024 (ExcCode 09)
<4>[136515.651219] PrId  : 0001992f (MIPS 1004Kc)
//...
<4>[136515.651543] Process ksoftirqd/1 (pid: 11, threadinfo=8fc80000, task=8fc54850, tls=00000000)
<4>[136515.651650] Stack : 00000014 83aa9000 81d5a368 00000000 8d4d0000 802a1138 00000002 87f75990
<4>[136515.651650] 	  8bff0008 8d49e504 81d5a368 8d49d9b0 8d4d0000 00000000 00000014 8bff0008
<4>[136515.651650] 	  8bff0008 8d41d1d8 00000001 00000001 00540046 00000001 00000000 8fc81b74
<4>[136515.651650] 	  8bff0008 8d4b338c 8d609b78 8d603f88 00000038 8d49e390 8f1050c0 000003d0
<4>[136515.651650] 	  8d609b78 0000004e 00000000 87f75990 81d5a368 8fc81c6c 00000001 00000001
<4>[136515.651650] 	  ...
<4>[136515.651656] Call Trace:
<4>[136515.651670] [<800a2ec4>] __free_page_frag+0x4c/0x9c
<4>[136515.651710] [<802a1138>] __kfree_skb+0x28/0xb0
<4>[136515.652056] [<8d49d9b0>] TestPacketDestory+0x2c/0x5c [testmod]
<4>[136515.652256] [<8d41d1d8>] TestProcess+0x130/0x1394 [testmod]
<4>[136515.652450] [<8d4bf8f4>] TestPacketIn+0x77c/0xfe0 [testmod]
```

从这个栈上可以看到是业务模块释放skb时，触发了double free BUG。
开始走查业务代码，检查是否有逻辑导致重复释放skb。仔细检查了一遍，没有发现重复释放代码逻辑。
因为不知道第一次释放时在哪个地方，只知道这次释放时触发了double free，无法定位问题，思路陷入僵局。

观察crash log发现crash前都有如下打印。

```c
br0: port 2(eth0.1) entered disabled state
```
推测跟接口的up/down有关。于是尝试ifconfig eth0.1 down/up，发现概率性触发crash。
但crash栈并未提供到太多帮助。

```c
<4>[  239.771433] [<80043ba4>] put_pid+0x8/0x58
<4>[  239.775430] [<803331ec>] unix_destruct_scm+0x44/0x70
<4>[  239.780392] [<8029bb5c>] skb_release_head_state+0x68/0xac
<4>[  239.785883] [<8029ec48>] __kfree_skb+0x14/0xb0
<4>[  239.790429] [<802736b0>] fe_txd_unmap.isra.0+0x318/0x38c
<4>[  239.795790] [<80274eb4>] fe_poll+0x12c/0x710
<4>[  239.800156] [<802afb28>] net_rx_action+0x130/0x2e0
<4>[  239.805019] [<8002fce8>] __do_softirq+0x294/0x2e0
<4>[  239.809767] [<8002fd6c>] run_ksoftirqd+0x38/0x6c
<4>[  239.814448] [<800488f0>] smpboot_thread_fn+0x18c/0x1bc
<4>[  239.819647] [<80045fd8>] kthread+0xd8/0xec
<4>[  239.823797] [<80005478>] ret_from_kernel_thread+0x14/0x1c
```

后发现ifconfig br0 down/up，100%触发crash。部分crash栈比较有分析意义。比如下面这个异常栈。

```c

<6>[77967.751362] br0: port 2(eth0.1) entered disabled state
<6>[77967.751614] br0: port 1(eth0.2) entered disabled state
<0>[77967.758186] page:810cff20 count:0 mapcount:0 mapping:  (null) index:0x0
<0>[77967.758210] flags: 0x0()
<1>[77967.758271] page dumped because: VM_BUG_ON_PAGE(({ union { typeof((&page->_count)->counter) __val; char __c[1]; } __u; if (1) __read_once_size(&((&page->_count)->counter), __u.__c, sizeof((&page->_count)->counter)); else __read_once_size_nocheck(&((&page->_count)->counter), __u.__c, sizeof((&page->_count)->counter)); __u.__val; }) == 0)
<4>[77967.758300] Kernel bug detected[#1]:
<4>[77967.758376] CPU: 3 PID: 19 Comm: ksoftirqd/3 Not tainted 4.4.167 #0
<4>[77967.758414] task: 8fc574d0 ti: 8fca0000 task.ti: 8fca0000
<4>[77967.758431] $ 0   : 00000000 00000001 00000000 00010000
<4>[77967.758456] $ 4   : 8047b634 00000001 0000da90 00000000
<4>[77967.758527] $ 8   : 00000002 804e7c40 00000000 297d203b
<4>[77967.758569] $12   : 00000001 8c148140 00000000 00000000
<4>[77967.758587] $16   : 877f16c0 8e937048 8e937000 877f16c0
<4>[77967.758603] $20   : 02080020 00000008 8e93705c 00000000
<4>[77967.758621] $24   : 00000002 8cc1d0a8                  
<4>[77967.758639] $28   : 8fca0000 8fca1bd8 02080020 800a2ec4
<4>[77967.758644] Hi    : 0016122a
<4>[77967.758649] Lo    : 62f04399
<4>[77967.758709] epc   : 800a2ec4 __free_page_frag+0x4c/0x9c
<4>[77967.758720] ra    : 800a2ec4 __free_page_frag+0x4c/0x9c
<4>[77967.758735] Status: 11007c03	KERNEL EXL IE 
<4>[77967.758741] Cause : 50800024 (ExcCode 09)
<4>[77967.758749] PrId  : 0001992f (MIPS 1004Kc)
//...
<4>[77967.759201] Process ksoftirqd/3 (pid: 19, threadinfo=8fca0000, task=8fc574d0, tls=00000000)
<4>[77967.759334] Stack : 8e621800 877f16c0 8e937048 8e937000 877f16c0 802a1138 c179e000 00000000
<4>[77967.759334] 	  00000000 00000000 877f16c0 802e50fc 00000000 00000000 00000000 00000000
<4>[77967.759334] 	  00000000 00000000 00000000 00000000 00000000 65000a0a 877f16c0 00000054
<4>[77967.759334] 	  8e621800 8e937000 80475cfc 8e937048 8e937000 802b0720 00000000 00000000
<4>[77967.759334] 	  00000000 00000000 80474888 80474b94 01000001 00000000 877f16c0 8ee32600
<4>[77967.759334] 	  ...
<4>[77967.759342] Call Trace:
<4>[77967.759374] [<800a2ec4>] __free_page_frag+0x4c/0x9c
<4>[77967.759426] [<802a1138>] __kfree_skb+0x28/0xb0
<4>[77967.759453] [<802e50fc>] ip_rcv+0x230/0x320
<4>[77967.759501] [<802b0720>] __netif_receive_skb_core+0x63c/0x6f4
<4>[77967.759522] [<802b1284>] netif_receive_skb_internal+0x74/0xb8
<4>[77967.759543] [<8037d0b8>] br_pass_frame_up+0xc0/0x130
<4>[77967.759557] [<8037d92c>] br_handle_frame+0x2c8/0x414
<4>[77967.759574] [<802b049c>] __netif_receive_skb_core+0x3b8/0x6f4
<4>[77967.759587] [<802b22d8>] process_backlog+0xac/0x184
<4>[77967.759605] [<802b207c>] net_rx_action+0x130/0x2e0
<4>[77967.759642] [<8002fd44>] __do_softirq+0x294/0x2e0
<4>[77967.759657] [<8002fdc8>] run_ksoftirqd+0x38/0x6c
<4>[77967.759688] [<8004894c>] smpboot_thread_fn+0x18c/0x1bc
<4>[77967.759722] [<80046034>] kthread+0xd8/0xec
<4>[77967.759738] [<80005478>] ret_from_kernel_thread+0x14/0x1c
```
一是这里也是释放skb导致触发double free check。和我们业务逻辑中释放skb导致double free check一样。
二是我们业务模块会注册bridge hook，在br_handle_frame中通过NF_HOOK将数据包取到，进入我们的业务处理逻辑，并返回NF_STOLEN，正常情况下不会继续进入到br_pass_frame_up逻辑。

基于以上两点怀疑有数据包被我们业务模块处理过程中释放，然后又继续走br_pass_frame_up，到ip_rcv逻辑时，又释放了一次，导致double free。

其实到这里基本接近答案了，但是因为两个点导致走了一些弯路。
1. 想当然的认为能走到我们处理逻辑里，必定会返回NF_STOLEN给NF_HOOK，想当然认为，系统不会继续进行到br_pass_frame_up。
2. 使用linux内核源码去分析问题，而不是使用openwrt编译的真正源码（编译环境同事在维护，想当然认为代码一样，就没有去对比）

基于这两个困住自己的点，让我又继续增加debug信息，将我们业务代码中释放skb的API，增加了dump_skb和dump_stack处理。
满屏的打印，让人绝望，但是确实抓到业务模块释放skb的信息和调用栈。但crash时ip_rcv释放的skb信息没有，于是修改内核代码，增加dump_skb和dump_stack处理。
继续调试，在多次的crash之后，终于抓到ip_rcv释放的skb和业务模块释放的skb是同一个skb。

```c
[  266.493265] dump skb: skb:8e5b2180 head:8d7bb0e0 data:8d7bb134 tail:8d7bb174 end:8d7bb740 len:64 data_len:0 truesize:2016 protocol:2048
//...
[  266.570564] Call Trace:
[  266.573012] [<80016b90>] show_stack+0x54/0x88
[  266.577389] [<801bf164>] dump_stack+0x84/0xbc
[  266.582056] [<8d09dcf4>] TestPacketDestory+0x270/0x2bc [testmod]
[  266.588583] [<8d01d984>] TestDpProcess+0x8dc/0x1494 [testmod]
[  266.594323] [<8d0bfca4>] TestPacketIn+0x77c/0x1008 [testmod]

[  266.735366] skbuff: [ip_rcv:464]DUMP: skb=8e5b2180 pkt_type=3 head=8d7bb0e0 data=8d7bb134 tail=8d7bb174 end=8d7bb740 
...
[  266.643709] Call Trace:
[  266.646154] [<80016b90>] show_stack+0x54/0x88
[  266.650506] [<801bf164>] dump_stack+0x84/0xbc
[  266.654878] [<8029dbf4>] skb_dump+0x74/0x180
[  266.659144] [<802e2e84>] ip_rcv+0x240/0x33c
[  266.663340] [<802ae3e4>] __netif_receive_skb_core+0x63c/0x6f4
[  266.669064] [<802aef48>] netif_receive_skb_internal+0x74/0xb8
[  266.674796] [<8037adb4>] br_pass_frame_up+0xc0/0x130
[  266.679752] [<8037b638>] br_handle_frame+0x2c8/0x414
[  266.684723] [<802ae160>] __netif_receive_skb_core+0x3b8/0x6f4
```

此时开始怀疑返回NF_STOLEN后系统继续向下处理了，于是要到了编译环境，查看br_handle_frame,发现多了一部分代码。

```c
//...省略无关
switch (p->state) {
case BR_STATE_DISABLED:
	if (ether_addr_equal(p->br->dev->dev_addr, dest))
		skb->pkt_type = PACKET_HOST;
	if (NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING, dev_net(skb->dev), NULL, skb, skb->dev, NULL,
		br_handle_local_finish))
		break;
	BR_INPUT_SKB_CB(skb)->brdev = p->br->dev;
	br_pass_frame_up(skb);
	break;
case BR_STATE_FORWARDING:
//...省略无关
```
怎么会多了一个 BR_STATE_DISABLED的case，怪不得接口down的时候会crash，进入了这个逻辑，我们再往下看，NF_HOOK处理，这次认真看了下，当我们的处理返回NF_STOLEN后，最终NF_HOOK返回值是0，所以就会进入到下面的br_pass_frame_up(skb)，继续向后处理。
问题就是出在这里，正常来说我们STOLEN后，表示数据包由我们处置，当遇到错误后，将数据包释放。
但在这个逻辑下，被释放的skb会被继续处理，到达ip_rcv后因为type=OTHER_HOST，所以被释放，导致了重复释放。
由于skb的重复释放会连带skb cache和数据包buffer，所以产生了各种各样的异常crash。

原因找到了，那么这块造成问题的代码哪里来的呢？
经过查找，这是一个patch代码，

```c
wangyao@appex:/home/work_7621_mips_src/openwrt$ cat target/linux/generic/patches-4.4/120-bridge_allow_receiption_on_disabled_port.patch
From: Stephen Hemminger <stephen@networkplumber.org>
Subject: bridge: allow receiption on disabled port

When an ethernet device is enslaved to a bridge, and the bridge STP
detects loss of carrier (or operational state down), then normally
packet receiption is blocked.

This breaks control applications like WPA which maybe expecting to
receive packets to negotiate to bring link up. The bridge needs to
block forwarding packets from these disabled ports, but there is no
hard requirement to not allow local packet delivery.

Signed-off-by: Stephen Hemminger <stephen@networkplumber.org>
Signed-off-by: Felix Fietkau <nbd@nbd.name>

-- a/net/bridge/br_input.c
+++ b/net/bridge/br_input.c
@@ -218,11 +218,13 @@ EXPORT_SYMBOL_GPL(br_handle_frame_finish
 static int br_handle_local_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
 {
         struct net_bridge_port *p = br_port_get_rcu(skb->dev);
-       u16 vid = 0;
+       if (p->state != BR_STATE_DISABLED) {
+               u16 vid = 0;
 
-       /* check if vlan is allowed, to avoid spoofing */
-       if (p->flags & BR_LEARNING && br_should_learn(p, skb, &vid))
-               br_fdb_update(p->br, p, eth_hdr(skb)->h_source, vid, false);
+               /* check if vlan is allowed, to avoid spoofing */
+               if (p->flags & BR_LEARNING && br_should_learn(p, skb, &vid))
+                       br_fdb_update(p->br, p, eth_hdr(skb)->h_source, vid, false);
+       }
        return 0;        /* process further */
 }

@@ -297,6 +299,18 @@ rx_handler_result_t br_handle_frame(stru

 forward:
        switch (p->state) {
+       case BR_STATE_DISABLED:
+               if (ether_addr_equal(p->br->dev->dev_addr, dest))
+                       skb->pkt_type = PACKET_HOST;
+
+               if (NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING, dev_net(skb->dev), NULL, skb, skb->dev, NULL,
+                       br_handle_local_finish))
+                       break;
+
+               BR_INPUT_SKB_CB(skb)->brdev = p->br->dev;
+               br_pass_frame_up(skb);
+               break;
+
        case BR_STATE_FORWARDING:
                rhook = rcu_dereference(br_should_route_hook);
                if (rhook) {

```

看描述，初衷时为了在DISABLED状态下依然可以接收发向本机的数据包。但是却引入了这么个bug，害苦我了。

将此patch的修改回退。出包验证，一切OK。纷繁嘈杂的世界又重归于宁静！

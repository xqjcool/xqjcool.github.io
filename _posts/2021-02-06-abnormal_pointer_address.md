---
title: "[Crash Analysis] Mysterious Pointer Address Overwritten"
date: 2021-02-06
---

最近遇到多次设备"general protection fault: 0000 [#1] SMP"错误。业务代码多个位置位置都出现过，甚至内核代码中都出现过，给定位带来很大难度，随着掌握的信息越来越多，最终将问题根源找到。前后大概折磨了一周多，直到最终定位才松了口气。

最初几次是crash在Diff函数中，下面是其中一个crash栈信息：

```bash
//省略无关
 #6 [ffff88008651fb30] general_protection at ffffffff8168f1a8
    [exception RIP: _T_Diff+193]
    RIP: ffffffffa0b6cea1  RSP: ffff88008651fbe0  RFLAGS: 00010206
    RAX: ffffc9003756b978  RBX: 00008fcd00000000  RCX: ffffc90037553010
    RDX: 0000000000000000  RSI: ffff880062b4a428  RDI: 0000000000000010
    RBP: ffff88008651fc28   R8: 0000000000000000   R9: 000000ffffffffff
    R10: 000000180075a8c0  R11: ffffea0004e56200  R12: ffff880062b4a428
    R13: ffffc90035503010  R14: ffff880062b4a428  R15: 0000000000000001
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88008651fc30] T_Diff at ffffffffa0b657ae [testmod]
 #8 [ffff88008651fd00] T_DiffByName at ffffffffa0b684fb [testmod]
//省略无关
crash> dis _T_Diff+193
0xffffffffa0b6cea1 <_T_Diff+193>:  cmpb   $0x0,0x11(%rbx)

crash> dis -r _T_Diff+193
0xffffffffa0b6ce80 <_T_Diff+160>:  mov    -0x30(%rbp),%eax
0xffffffffa0b6ce83 <_T_Diff+163>:  mov    -0x40(%rbp),%rcx
0xffffffffa0b6ce87 <_T_Diff+167>:  add    $0x1,%rax
0xffffffffa0b6ce8b <_T_Diff+171>:  shl    $0x4,%rax
0xffffffffa0b6ce8f <_T_Diff+175>:  lea    0x8(%rcx,%rax,1),%rax
0xffffffffa0b6ce94 <_T_Diff+180>:  mov    0x8(%rax),%rbx
0xffffffffa0b6ce98 <_T_Diff+184>:  test   %rbx,%rbx
0xffffffffa0b6ce9b <_T_Diff+187>:  je     0xffffffffa0b6cf20 <_T_Diff+320>
0xffffffffa0b6cea1 <_T_Diff+193>:  cmpb   $0x0,0x11(%rbx)

crash> rd ffffc9003756b978 4
ffffc9003756b978:  6488000100000000 00008fcd00000000   .......d........
ffffc9003756b988:  0000000000000000 0000000000000000   ................
```
RBX是结构中的一个地址指针，00008fcd00000000 ，从值上看像是结构创建后没有初始化。查看代码，结构初始化清零操作都完成了。指针释放后的清零操作也有。

随后怀疑是溢出导致，检查该位置相关操作代码，不存在溢出覆写可能，这块也排除了。

初步定位陷入僵局。
这时另一个设备也出现异常地址访问crash。

```bash
//省略无关
 #6 [ffff88013fb03800] general_protection at ffffffff8168f1a8
    [exception RIP: memcmp+9]
    RIP: ffffffff81321699  RSP: ffff88013fb038b8  RFLAGS: 00010206
    RAX: 00000000ffffffc0  RBX: ffff880017268000  RCX: 0000000000000074
    RDX: 0000000000000018  RSI: ffff880017268000  RDI: 26ff880066648818
    RBP: ffff88013fb038b8   R8: 0000000000000003   R9: 0000000000000000
    R10: 0000000000000000  R11: ffff880017268014  R12: 0000000000000000
    R13: 26ff880066648808  R14: 0000000000000000  R15: ffff880017268000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88013fb038c0] T_Lookup at ffffffffa0a53919 [testmod]
 #8 [ffff88013fb03978] T_Process at ffffffffa0a4f0f2 [testmod]
//省略部分
```

这个crash栈上，明显看到 RDI: 26ff880066**6488**18是一个地址，但是值26ff880066648818明显可以看出来最高byte错误，本应是0xff，却被改写成0x26。从而导致地址访问保护错误crash。

下面这个是发生在系统代码中的crash。

```bash
 #6 [ffff88009f653bc0] general_protection at ffffffff8168f1a8
    [exception RIP: update_queue+129]
    RIP: ffffffff81293d01  RSP: ffff88009f653c78  RFLAGS: 00010246
    RAX: ffff8801f9ae6488  RBX: 0000000000000001  RCX: 0000000000000000
    RDX: ffff88009f653d60  RSI: 00000000ffffffff  RDI: ffff8801f9ae6440
    RBP: ffff88009f653cc8   R8: ffff8801f9ae64c0   R9: 0000000000000001
    R10: ffff8801f9ae6440  R11: ffff880097fc0fb0  R12: ffff8801f9ae6440
    R13: 00000000ffffffff  R14: 287b8801f9ae6488  R15: ffffffffffffffc0
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88009f653cd0] do_smart_update at ffffffff8129441f
 #8 [ffff88009f653d10] SYSC_semtimedop at ffffffff81294e3a
 #9 [ffff88009f653f60] sys_semtimedop at ffffffff812961fe
 //省略无关
 crash> dis -l  update_queue+129
/usr/src/debug/kernel-3.10.0-514.26.2.el7/linux-3.10.0-514.26.2.el7.x86_64/ipc/sem.c: 884
0xffffffff81293d01 <update_queue+129>:  mov    (%r14),%rbx
```
查看R14: 287b8801f9ae**6488**，也是明显的地址高位2个byte被改写，应该是0xffff，被改写成0x287b。


基本上都是结构中地址指针中某些byte被改写错误导致。尤其是在系统处理中出现异常地址错误，极可能是我们业务代码导致，于是开始怀疑近期合入代码中存在BUG，会随机修改某个位置的值。如果恰好改的位置是地址指针，在随后访问时就会造成crash。

因为不易复现，且位置随机，从目前crash栈上无法推测造成问题的代码位置。只能一边以问题第一次出现的时间点向前进行代码review，尝试找出问题代码；一边对现有的crash进行归纳，尝试找出一些线索。

随后又出现一次crash，出错位置在更新pppoe payload长度上，而该设备并没有开启pppoe。继续分析发现，代码在判断上存在错误，强行将tcp payload解析为自定义tunnel结构去访问。当tcp为普通的ack报文时，是没有payload的。这样当作自定义tunnel结构访问就会得到错误的随机的内容。然后根据错误去进行偏移，得到pppoe头，再赋值，就会改写内存中其他结构的值，这就是上面那些crash的原因。

```bash
[1161061.246479] BUG: unable to handle kernel paging request at ffff8800b9c6b95a
[1161061.268049] IP: [<ffffffffa09fbffe>] _T_BaseSndPkt+0x5ce/0xa60 [testmod]
[1161061.290902] PGD 1fa3067 PUD 13fffe067 PMD 13fffc067 PTE 0
[1161061.307848] Oops: 0000 [#1] SMP 
//省略部分
[1161061.534210]  fjes i2c_hid sdhci_acpi i2c_core sdhci mmc_core dm_mirror dm_region_hash dm_log dm_mod nf_conntrack [last unloaded: testmod]
[1161061.567697] CPU: 0 PID: 3 Comm: ksoftirqd/0 Tainted: G           OE  ------------   3.10.0-514.26.2.el7.x86_64 #1
[1161061.599105] Hardware name: YanRay B1904/B1904, BIOS 5.6.5 03/20/2019
[1161061.618750] task: ffff880139af9f60 ti: ffff880139b74000 task.ti: ffff880139b74000
[1161061.641784] RIP: 0010:[<ffffffffa09fbffe>]  [<ffffffffa09fbffe>] _T_BaseSndPkt+0x5ce/0xa60 [testmod]
[1161061.671974] RSP: 0018:ffff880139b779b8  EFLAGS: 00010296
[1161061.688479] RAX: ffff8800b9c6c0b0 RBX: ffff8800aadd7f78 RCX: ffff8800b9c6b94e
[1161061.710479] RDX: 0000000000000762 RSI: 00000000ace8521f RDI: 00000000ac1813fd
[1161061.732484] RBP: ffff880139b77a60 R08: 0000000000000000 R09: 0000000000000006
[1161061.754489] R10: 0000000000000088 R11: 0000000000000000 R12: ffff88010140ac00
[1161061.776494] R13: ffff880139b77b10 R14: ffff8800b9c6c09c R15: 0000000000000001
[1161061.798501] FS:  0000000000000000(0000) GS:ffff88013fc00000(0000) knlGS:0000000000000000
[1161061.823382] CS:  0010 DS: 0000 ES: 0000 CR0: 000000008005003b
[1161061.841194] CR2: ffff8800b9c6b95a CR3: 000000013fda2000 CR4: 00000000001007f0
[1161061.863188] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[1161061.885193] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
[1161061.907199] Stack:
[1161061.913785]  ffff880139b77a00 e3e3ddc0a09fc79d 0000005b00000029 0000000000000000
[1161061.936733]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[1161061.959677]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[1161061.982618] Call Trace:
[1161061.990541]  [<ffffffffa09fc5c6>] T_BaseSndPkt+0xb6/0xe0 [testmod]
[1161062.011000]  [<ffffffffa0a1234b>] T_SndPkt+0xdb/0x170 [testmod]
```

tcp ack报文很常见，为何不是必现呢。这是因为这块代码还有个判断条件，当满足以下条件时
(ethHdr->h_proto == __constant_htons(ETH_P_PPP_SES))
才会改写。也就是说随机偏移到的内存位置，特定的字节要满足 0x6488才行，这个条件足够苛刻，所以问题才没那么容易出现，但是一旦出现又很难定位。（感兴趣的读者可以回看一下，错误位置附近是不是都有0x6488这个值 :-) ）

后续如果碰到这种地址指针中间被改写，大概率是代码问题导致随机读写异常。一方面要看异常内存前后有没有什么特定数值；另一方面review近期的提交，查看有没有异常读写代码。


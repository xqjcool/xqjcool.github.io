---
title: "[crash analysis][mips]CPU 0 Unable to handle kernel paging request at virtual address"
date: 2020-02-29
---
系统平台是mips 32位架构，openwrt定制系统。产品模块加载到内核中工作。因为是低端产品，硬件配置简陋，所以系统异常crash后无法保存coredump，只有串口记录crash log供分析定位。
    以下是最近遇到的一个crash：
	

```powershell
[  293.152901] CPU 0 Unable to handle kernel paging request at virtual address 235b5cf4, epc == 85cb1b68, ra == 85cb1b00

[  293.163955] Oops[#1]:

[  293.166309] CPU: 0 PID: 3 Comm: ksoftirqd/0 Not tainted 4.4.59 #0

[  293.172605] task: 87c34a10 ti: 87c44000 task.ti: 87c44000

[  293.178180] $ 0   : 00000000 00000001 235b5cf4 235b5cf7

[  293.183617] $ 4   : 87c45a4c 87f39052 00000000 87f39068

[  293.189047] $ 8   : 00000000 00000000 00000000 6985f312

[  293.194477] $12   : f6a9c845 80000052 00000000 00000000

[  293.199906] $16   : 872c3f00 80433870 00000000 00000000

[  293.205336] $20   : 00000002 0000076c 00000011 00000000

[  293.210766] $24   : 00000002 8007d508                  

[  293.216196] $28   : 87c44000 87c45a30 871c94c8 85cb1b00

[  293.221627] Hi    : 00000003

[  293.224601] Lo    : 3e8752f3

[  293.227910] epc   : 85cb1b68 FuncBrOutput+0x304/0x484 [funcmod]

[  293.234587] ra    : 85cb1b00 FuncBrOutput+0x29c/0x484 [funcmod]

[  293.240972] Status: 1000fc03 KERNEL EXL IE 

[  293.245327] Cause : 00800008 (ExcCode 02)

[  293.249466] BadVA : 235b5cf4

[  293.252442] PrId  : 00019374 (MIPS 24Kc)

...	//省略无关信息

[  293.333616] Process ksoftirqd/0 (pid: 3, threadinfo=87c44000, task=87c34a10, tls=00000000)

[  293.342146] Stack : 00007c7f 00000001 00000000 85c186a8 85db39e4 800b7484 85da241c 85da2418

          00000000 000000ff 87db5800 5e58a4d0 235b5cf7 00000000 87c45ab8 06a906a9

          0a010494 8b09bb77 87156c08 00000000 862edb48 85c175ac 87156c08 862edb48

          84bb2c88 00000000 00007c7f 85c3cd20 87c45aef 872c3f00 00000000 00000001

          862eda88 85cb7048 03f0a740 5e58a4d0 00007c7f 00000000 00000170 8a422502

          ...

[  293.379137] Call Trace:

[  293.381961] [<85cb1b68>] FuncBrOutput+0x304/0x484 [funcmod]

```
   很明显这是异常地址访问导致的系统崩溃，对照Call Trace可知问题出在FuncBrOutput函数中，但是这个函数相对较长，无法快速定位到行数。
    于是只好将对应函数的反汇编取出，通过汇编代码和源码来确定问题原因。
	
```powershell
000b1864 <FuncBrOutput>:
   b1864: 27bdff78  addiu sp,sp,-136
   b1868: afb00074  sw s0,116(sp)
   b186c: afbf0084  sw ra,132(sp)
   b1870: afb30080  sw s3,128(sp)
   b1874: afb2007c  sw s2,124(sp)
   b1878: afb10078  sw s1,120(sp)
   b187c: 8cc200a0  lw v0,160(a2)
   b1880: 94420002  lhu v0,2(v0)
   b1884: 14400007  bnez v0,b18a4 <FuncBrOutput+0x40>
   b1888: 00c08025  move s0,a2		//s0=a2，即将参数3 skb指针的值放在s0中
...  //省略无关
   b1aa4: 00001025  move v0,zero
   b1aa8: 24060014  li a2,20
   b1aac: 26040018  addiu a0,s0,24
   b1ab0: 0c000000  jal 0 <...>
   b1ab4: 00002825  move a1,zero
   b1ab8: 9602007e  lhu v0,126(s0)
   b1abc: 3c040000  lui a0,0x0
   b1ac0: 24840a10  addiu a0,a0,2576
   b1ac4: a4820018  sh v0,24(a0)
   b1ac8: 9602007c  lhu v0,124(s0)
   b1acc: 96060094  lhu a2,148(s0)
   b1ad0: a482001a  sh v0,26(a0)
   b1ad4: 34028100  li v0,0x8100
   b1ad8: 14c20056  bne a2,v0,b1c34 <FuncBrOutput+0x3d0>
   b1adc: 38c68864  xori a2,a2,0x8864
   b1ae0: 24060004  li a2,4
   b1ae4: 30c600ff  andi a2,a2,0xff
   b1ae8: 8e0500a8  lw a1,168(s0)
   b1aec: a0860016  sb a2,22(a0)
   b1af0: 24c6000e  addiu a2,a2,14
   b1af4: 00a62823  subu a1,a1,a2
   b1af8: 0c000000  jal 0 <...>
   b1afc: a0860017  sb a2,23(a0)
   b1b00: 8e020014  lw v0,20(s0)	//v0=[s0+20],即v0保存了skb->dev
   b1b04: 27a4001c  addiu a0,sp,28	//a0 = sp + 28
   b1b08: 8e030010  lw v1,16(s0)
   b1b0c: afa20028  sw v0,40(sp)	//[sp+40]=v0,即将skb->dev存到了本地变量中
   b1b10: 34820001  ori v0,a0,0x1
...  //省略无关
   b1b5c: 8c830014  lw v1,20(a0)    //v1 = [a0 + 20]
   b1b60: 2402fffc  li v0,-4     //v0 = -4
   b1b64: 00431024  and v0,v0,v1    //v0 = v0 & v1
   b1b68: 8c420000  lw v0,0(v0)    //出错位置 v0地址异常
   b1b6c: 30420004  andi v0,v0,0x4
   b1b70: 14400005  bnez v0,b1b88 <FuncBrOutput+0x324>
...  //省略无关

```
   汇编代码太长，我将无关代码省略。
    首先我们找到出错位置 85cb1b68 也就是这里的 b1b68，看指令是取v0指向内存时导致错误。从Call Trace log中我们也看到了v0($3寄存器)值为235b5cf4，和 “Unable to handle kernel paging request at virtual address 235b5cf4”一致。
反推前几条指令，可以看到这是将[a0 + 20]的值掩掉最后2个bit，作为地址进行访问。
    a0从哪里来的呢，我们继续向前找，大部分指令包括省略的都不会改变a0的值，直到b1b04这条指令，设置a0=sp+28。地址在栈中，这说明a0指向函数本地变量。 
    汇编上看[a0+20]并未设置，但是却直接访问操作了，肯定会出问题的。至此基本原因找到了。但是这些操作到底对应那些具体代码呢？
    在函数开始我们看到 move s0，a2， a2是我们的入参3，是skb。s0在之后的操作中没有改变，这为我们分析带来了便利。
    对照b1b00和b1b0c我们可以知道这是个skb->dev的赋值操作，位置是sp+40, 以a0为基准的话就是 a0+12，简言之就是将skb->dev,存到a0代表的临时变量的offset 12的位置。对照代码我们找到了tmpEntry。这是dst_entry结构，而出错位置是访问_metrics。


```c
struct dst_entry temEntry;
...	//省略无关
#ifndef OPENWRT
        dst_init(&temEntry, &dstops, NULL, 1,
             DST_OBSOLETE_NONE, DST_NOCOUNT);
#endif

...	//省略无关
	temEntry.dev = skb->dev;
...	//省略无关
	skb_dst_set_noref(skb, &temEntry);

	ret = func_ip_fragment(net, sk, skb, func_push_frag_xmit_sk);
```
   函数将tempEntry通过skb_dst_set_noref设置到skb，随后在func_ip_fragment中会调用ip_skb_dst_mtu来获取mtu。在这个函数的层层调用，最终会获取dst的_metrics然后对最后两个bit置0，并以这个值作为地址访问。
    ip_skb_dst_mtu-->ip_dst_mtu_maybe_forward-->dst_metric_locked-->dst_metric-->dst_metric_raw-->DST_METRICS_PTR-->__DST_METRICS_PTR
	

```c
#define DST_METRICS_FLAGS  0x3UL
#define __DST_METRICS_PTR(Y) \
 ((u32 *)((Y) & ~DST_METRICS_FLAGS))
#define DST_METRICS_PTR(X) __DST_METRICS_PTR((X)->_metrics)
```
   至此问题定位完成，在OPENWRT平台下dst_init没有执行，导致temEntry中的变量值为随机值，后续获取mtu导致造作_metrics时，导致异常地址访问。
 

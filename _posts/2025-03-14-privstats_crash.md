---
title: "The use-after-free crash issue that had been troubling us for over a year has finally been identified and resolved!"
date: 2025-03-14
---
# 一、 问题初现

那是一年多前，客户上报了一个设备crash问题，当时拿到coredump后，先是发现_skb_refdst异常。

## coredump 1 号

### 异常栈分析 
异常发丧个在CPU0上。“PANIC: "general protection fault: 0000 [#1] SMP PTI"”提示异常地址访问，导致crash。

```c
 #5 [ffff88827ec03a70] general_protection at ffffffff80c012c5
    [exception RIP: ip_route_input_rcu+507]
    RIP: ffffffff8091c63b  RSP: ffff88827ec03b20  RFLAGS: 00010246
    RAX: ffff888150ba6b0e  RBX: ffff888181b60500  RCX: 000000000000c910
    RDX: 9ac14df3f281abd3  RSI: 0000000000000006  RDI: ffff888150ba6b0e
    RBP: ffff88827ec03bc8   R8: 0000000000000000   R9: 000000000000bb01
    R10: 0000000000000028  R11: ffff88827ec03be0  R12: ffff8882329f5000
    R13: 000000002ea3d9ac  R14: 00000000c8b2150a  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #6 [ffff88827ec03bd0] ip_route_input_noref at ffffffff8091d53b
 #7 [ffff88827ec03c60] ip_rcv_finish at ffffffff8091fe5b
 #8 [ffff88827ec03c98] ip_rcv at ffffffff80921741
 #9 [ffff88827ec03d28] __netif_receive_skb_core at ffffffff807fd0ba
#10 [ffff88827ec03db0] __netif_receive_skb at ffffffff807ff868
#11 [ffff88827ec03dd0] netif_receive_skb_internal at ffffffff8080325f
#12 [ffff88827ec03e28] napi_gro_receive at ffffffff80803f85
#13 [ffff88827ec03e50] igb_poll at ffffffff806a17f1
#14 [ffff88827ec03ec8] net_rx_action at ffffffff80803825
#15 [ffff88827ec03f40] __softirqentry_text_start at ffffffff80e000e2
#16 [ffff88827ec03fa8] irq_exit at ffffffff8025c191
#17 [ffff88827ec03fb8] do_IRQ at ffffffff80c0217a
```

查看反汇编

```c
crash> dis -r ip_route_input_rcu+507   
0xffffffff8091c440 <ip_route_input_rcu>:        nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff8091c445 <ip_route_input_rcu+5>:      push   %rbp
0xffffffff8091c446 <ip_route_input_rcu+6>:      mov    %r9,%r11
0xffffffff8091c449 <ip_route_input_rcu+9>:      xor    %r9d,%r9d
0xffffffff8091c44c <ip_route_input_rcu+12>:     mov    %rsp,%rbp
0xffffffff8091c44f <ip_route_input_rcu+15>:     push   %r15
0xffffffff8091c451 <ip_route_input_rcu+17>:     mov    %ecx,%r15d
0xffffffff8091c454 <ip_route_input_rcu+20>:     push   %r14
0xffffffff8091c456 <ip_route_input_rcu+22>:     mov    %edx,%r14d
0xffffffff8091c459 <ip_route_input_rcu+25>:     push   %r13
0xffffffff8091c45b <ip_route_input_rcu+27>:     mov    %esi,%r13d
0xffffffff8091c45e <ip_route_input_rcu+30>:     push   %r12
0xffffffff8091c460 <ip_route_input_rcu+32>:     mov    %r8,%r12
0xffffffff8091c463 <ip_route_input_rcu+35>:     xor    %r8d,%r8d
0xffffffff8091c466 <ip_route_input_rcu+38>:     push   %rbx
0xffffffff8091c467 <ip_route_input_rcu+39>:     mov    %rdi,%rbx	//rbx=rdi=skb=ffff888181b60500
//省略无关部分
0xffffffff8091c5fb <ip_route_input_rcu+443>:    mov    0x68(%rbx),%rax	//rax=_skb_refdst=0xffff888150ba6b0e
0xffffffff8091c5ff <ip_route_input_rcu+447>:    mov    %rax,%rdi				//rdi=_skb_refdst
0xffffffff8091c602 <ip_route_input_rcu+450>:    and    $0xfffffffffffffffe,%rdi			//rdi=dst
0xffffffff8091c606 <ip_route_input_rcu+454>:    je     0xffffffff8091ca36 <ip_route_input_rcu+1526>
0xffffffff8091c60c <ip_route_input_rcu+460>:    testb  $0x80,0x60(%rdi)
0xffffffff8091c610 <ip_route_input_rcu+464>:    je     0xffffffff8091c625 <ip_route_input_rcu+485>
0xffffffff8091c612 <ip_route_input_rcu+466>:    mov    0xa0(%rdi),%edx
0xffffffff8091c618 <ip_route_input_rcu+472>:    test   %edx,%edx
0xffffffff8091c61a <ip_route_input_rcu+474>:    jne    0xffffffff8091c625 <ip_route_input_rcu+485>
0xffffffff8091c61c <ip_route_input_rcu+476>:    lea    0xa8(%rdi),%rdx
0xffffffff8091c623 <ip_route_input_rcu+483>:    jmp    0xffffffff8091c638 <ip_route_input_rcu+504>
0xffffffff8091c625 <ip_route_input_rcu+485>:    mov    0x90(%rdi),%rdx	//rdx=dst_entry.lwstate=9ac14df3f281abd3 错误地址
0xffffffff8091c62c <ip_route_input_rcu+492>:    xor    %r8d,%r8d
0xffffffff8091c62f <ip_route_input_rcu+495>:    test   %rdx,%rdx
0xffffffff8091c632 <ip_route_input_rcu+498>:    je     0xffffffff8091c645 <ip_route_input_rcu+517>
0xffffffff8091c634 <ip_route_input_rcu+500>:    add    $0x30,%rdx
0xffffffff8091c638 <ip_route_input_rcu+504>:    xor    %r8d,%r8d
0xffffffff8091c63b <ip_route_input_rcu+507>:    testb  $0x1,0x49(%rdx)	//错误地址访问
```
这里的错误对应内核以下代码：
通过反汇编 skb=ffff88822eb79500
ash> sk_buff._skb_refdst -x ffff888181b60500
  _skb_refdst = 0xffff888150ba6b0e		//这已经是一个异常地址
通过寄存器可以知道 lwstate=0031872eba99ed1a
进而访问dst->lwtstate->type 时 触发异常地址访问。

```c
static inline struct ip_tunnel_info *skb_tunnel_info(struct sk_buff *skb)
{
        struct metadata_dst *md_dst = skb_metadata_dst(skb);
        struct dst_entry *dst;

        if (md_dst && md_dst->type == METADATA_IP_TUNNEL)
                return &md_dst->u.tun_info;

        dst = skb_dst(skb);
        if (dst && dst->lwtstate &&					//lwtstate=9ac14df3f281abd3 后面访问type异常
            (dst->lwtstate->type == LWTUNNEL_ENCAP_IP ||
             dst->lwtstate->type == LWTUNNEL_ENCAP_IP6))
                return lwt_tun_info(dst->lwtstate);

        return NULL;
}


```
造成问题的表面原因清除了，就是 _skb_refdst地址异常导致的。但是为什么地址会异常呢？
对照代码和coredump，对问题的产生原因，作了一些推测，然后又一一推翻了。
### 挖掘另一个核的栈信息
为了收集更多信息，就去查看另一个CPU的栈信息，发现CPU1也发生了异常地址访问。

```c
crash> bt -c 1
PID: 0        TASK: ffff888275a89a00  CPU: 1    COMMAND: "swapper/1"
 #0 [ffff88827ed03838] die at ffffffff8021cc8e
 #1 [ffff88827ed03868] do_general_protection at ffffffff8021ae4e
 #2 [ffff88827ed03890] general_protection at ffffffff80c012c5
    [exception RIP: ipv4_helper+109]
    RIP: ffffffff809a0f3d  RSP: ffff88827ed03940  RFLAGS: 00010206
    RAX: 0000000000000005  RBX: ffff88822e0f4480  RCX: 0000000000000003
    RDX: ffff88818139b500  RSI: 0000000000000014  RDI: ffff88822eb79500
    RBP: ffff88827ed03940   R8: 050a047063740300   R9: ffff888250540000
    R10: 0000000000000002  R11: 0000000000000004  R12: ffff88822eb79500
    R13: ffff88822e0f44a0  R14: 0000000000000004  R15: ffff888255372dc0
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #3 [ffff88827ed03948] nf_reinject at ffffffff8085445c
 #4 [ffff88827ed03990] mmq_dev_xmit at ffffffff8065fd6a
 #5 [ffff88827ed039b8] dev_hard_start_xmit at ffffffff80800dab
 #6 [ffff88827ed03a18] __mmq_nf_queue at ffffffff80660170
 #7 [ffff88827ed03a78] mmq_nf_queue at ffffffff806609f8
 #8 [ffff88827ed03ad0] nf_queue at ffffffff8085426e

```

查看对应的反汇编

```c
crash> dis -r ipv4_helper+109    
0xffffffff809a0ed0 <ipv4_helper>:       nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff809a0ed5 <ipv4_helper+5>:     mov    0x80(%rsi),%rdx	//sk_buff._nfct
0xffffffff809a0edc <ipv4_helper+12>:    mov    %edx,%ecx
0xffffffff809a0ede <ipv4_helper+14>:    and    $0x7,%ecx
0xffffffff809a0ee1 <ipv4_helper+17>:    and    $0xfffffffffffffff8,%rdx
0xffffffff809a0ee5 <ipv4_helper+21>:    je     0xffffffff809a0f48 <ipv4_helper+120>
0xffffffff809a0ee7 <ipv4_helper+23>:    cmp    $0x4,%ecx
0xffffffff809a0eea <ipv4_helper+26>:    je     0xffffffff809a0f48 <ipv4_helper+120>
0xffffffff809a0eec <ipv4_helper+28>:    mov    0xf0(%rdx),%rdi	//nf_conn.ext
0xffffffff809a0ef3 <ipv4_helper+35>:    mov    $0x1,%eax
0xffffffff809a0ef8 <ipv4_helper+40>:    test   %rdi,%rdi
0xffffffff809a0efb <ipv4_helper+43>:    je     0xffffffff809a0f4f <ipv4_helper+127>
0xffffffff809a0efd <ipv4_helper+45>:    movzwl 0x10(%rdi),%r8d	//offset[0]
0xffffffff809a0f02 <ipv4_helper+50>:    test   %r8w,%r8w
0xffffffff809a0f06 <ipv4_helper+54>:    je     0xffffffff809a0f4e <ipv4_helper+126>
0xffffffff809a0f08 <ipv4_helper+56>:    add    %r8,%rdi		//nf_conn_help
0xffffffff809a0f0b <ipv4_helper+59>:    je     0xffffffff809a0f50 <ipv4_helper+128>
0xffffffff809a0f0d <ipv4_helper+61>:    mov    (%rdi),%r8	//nf_conntrack_helper=050a047063740300
0xffffffff809a0f10 <ipv4_helper+64>:    test   %r8,%r8
0xffffffff809a0f13 <ipv4_helper+67>:    je     0xffffffff809a0f4f <ipv4_helper+127>
0xffffffff809a0f15 <ipv4_helper+69>:    push   %rbp
0xffffffff809a0f16 <ipv4_helper+70>:    movzwl 0xdc(%rsi),%eax
0xffffffff809a0f1d <ipv4_helper+77>:    mov    %rsi,%rdi	//skb
0xffffffff809a0f20 <ipv4_helper+80>:    add    0x118(%rsi),%rax
0xffffffff809a0f27 <ipv4_helper+87>:    mov    %rax,%rsi
0xffffffff809a0f2a <ipv4_helper+90>:    movzbl (%rax),%eax
0xffffffff809a0f2d <ipv4_helper+93>:    sub    0x120(%rdi),%rsi
0xffffffff809a0f34 <ipv4_helper+100>:   mov    %rsp,%rbp
0xffffffff809a0f37 <ipv4_helper+103>:   and    $0xf,%eax
0xffffffff809a0f3a <ipv4_helper+106>:   lea    (%rsi,%rax,4),%esi
0xffffffff809a0f3d <ipv4_helper+109>:   mov    0x68(%r8),%rax	//异常地址访问

```

对应代码

```c
static unsigned int ipv4_helper(void *priv,
                                struct sk_buff *skb,
                                const struct nf_hook_state *state)
{
        struct nf_conn *ct;
        enum ip_conntrack_info ctinfo;
        const struct nf_conn_help *help;
        const struct nf_conntrack_helper *helper;

        /* This is where we call the helper: as the packet goes out. */
        ct = nf_ct_get(skb, &ctinfo);
        if (!ct || ctinfo == IP_CT_RELATED_REPLY)
                return NF_ACCEPT;

        help = nfct_help(ct);
        if (!help)
                return NF_ACCEPT;

        /* rcu_read_lock()ed by nf_hook_thresh */
        helper = rcu_dereference(help->helper);
        if (!helper)
                return NF_ACCEPT;

        return helper->help(skb, skb_network_offset(skb) + ip_hdrlen(skb),
                            ct, ctinfo);	//helper=050a047063740300 错误地址 helper->help 异常地址访问。
}

```

skb=ffff88822eb79500
```c
crash> sk_buff._nfct -x ffff88822eb79500
  _nfct = 0xffff88818139b503,
crash> nf_conn.ext -x 0xffff88818139b500 
  ext = 0xffff88817fca8e00,
crash> nf_ct_ext -x 0xffff88817fca8e00 
struct nf_ct_ext {
  rcu = {
    next = 0x0,
    func = 0x0
  },
  offset = {0xfffd, 0x117, 0x0, 0x0, 0x40, 0x0, 0x0, 0x0, 0x9bcd, 0xd8, 0x30, 0x68, 0x0},
  len = 0x138,
  data = 0xffff88817fca8e2c ""
}
```
offset[0]=fffd 偏移异常，导致得到的 nf_conn_help地址异常，最终访问helper->help时，触发内核异常。

nf_ct_ext的offset， 明显有几个偏移值不正常， 目前长度是0x138，但是有 几个偏移值远超这界限。
  offset = {0xfffd, 0x117, 0x0, 0x0, 0x40, 0x0, 0x0, 0x0, 0x9bcd, 0xd8, 0x30, 0x68, 0x0},		--异常
//offset = {0x0, 0x118, 0x0, 0x0, 0x40, 0x0, 0x0, 0x0, 0x58, 0xd8, 0x30, 0x68, 0x0}, 			--对比正常offset
发现offset[0,1,8]都异常被改写了。这是造成 CPU0/CPU1 异常访问错误的更深层的原因。

但是这些内存被谁给改写了呢？暂时没有头绪。
### 大海捞针
既然 0xffff88817fca8e00 对应的内存被改写了，那么我们找下那些地方保存了这个地址，看看有没有新的发现。

```c
crash> search 0xffff88817fca8e00      
ffff888004208288: ffff88817fca8e00 
//省略部分
ffff888107c40188: ffff88817fca8e00 
ffff888107c40988: ffff88817fca8e00 
ffff888107ce7188: ffff88817fca8e00 
ffff888107ce71c0: ffff88817fca8e00 
ffff888107ce7388: ffff88817fca8e00 
//省略部分

```
逐个查看对应的地址，没有有效信息。直到ffff888107c40188，这是一个kmalloc-256分配的内存。

```c
crash> kmem ffff888107c40188 
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff888275803200      256       5402      6448    403     4k  kmalloc-256
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea00041f1000  ffff888107c40000     0     16         16     0
  FREE / [ALLOCATED]
  [ffff888107c40100]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00041f1000 107c40000                0        0  1 8000000000000100 slab
crash> rd ffff888107c40100 32 -s
ffff888107c40100:  ffff88824a0d9e80 0000000000000000 
ffff888107c40110:  0000000000000000 0000000000000040 
ffff888107c40120:  0068003000000058 0000000000d80000 
ffff888107c40130:  ffff88812ca6d200 0000000000000000 
ffff888107c40140:  0000000000000000 0000ffffffff0000 
ffff888107c40150:  0000000000000000 0000004b00000001 
ffff888107c40160:  000004000001c0f8 0000000000fb0003 
ffff888107c40170:  dead4ead00000000 00000000ffffffff 
ffff888107c40180:  ffffffffffffffff ffff88817fca8e00 
ffff888107c40190:  0000000000000000 0000000000000002 
ffff888107c401a0:  dead4ead00000000 00000000ffffffff 
ffff888107c401b0:  ffffffffffffffff 0000000000000000 
ffff888107c401c0:  ffff88817fca8600 ffff888108d3ba80 
ffff888107c401d0:  ffff888232ab2840 0000000000000000 
ffff888107c401e0:  0000000000000000 0000000000000000 
ffff888107c401f0:  0000004b00000000 0000000000000000 
crash> rd ffff88812ca6d200 -s 32
ffff88812ca6d200:  ffff888130041000 0000000000000000 
ffff88812ca6d210:  0000000000000000 0000000000000000 
ffff88812ca6d220:  ipv4_dst_ops     dst_default_metrics+1 
ffff88812ca6d230:  0000000427e0d300 ffff88812ca6d200 
ffff88812ca6d240:  0000000000000000 0000000000000000 
ffff88812ca6d250:  ip_forward       ip_output        
ffff88812ca6d260:  0000ffff00000600 0000000010070000 
ffff88812ca6d270:  dead4ead00000000 00000000ffffffff 
ffff88812ca6d280:  0000000000000035 0000000427e0cb94 
ffff88812ca6d290:  0000000000000000 0000000000000000 
ffff88812ca6d2a0:  0000000000000cd5 0000000000010001 
ffff88812ca6d2b0:  0000000000000000 00000000000000fb 
ffff88812ca6d2c0:  00000000000000fb ffff88812ca6d2c8 
ffff88812ca6d2d0:  ffff88812ca6d2c8 ffff888150b9ead8 
ffff88812ca6d2e0:  ffff88818b59e1d8 ffff88827ec44940 
ffff88812ca6d2f0:  0000000000000000 0000000000000000 

```
继续挖掘，发现offset 0x30的位置是一个dst_entry, 对比内核dst_entry相关代码，发现 这是一个 nf_ct_ext结构。而 0xffff88817fca8e00 位于其 nf_conn_priv结构中。

```c
crash> nf_ct_ext -x ffff888107c40100        
struct nf_ct_ext {
  rcu = {
    next = 0xffff88824a0d9e80,
    func = 0x0
  },
  offset = {0x0, 0x0, 0x0, 0x0, 0x40, 0x0, 0x0, 0x0, 0x58, 0x0, 0x30, 0x68, 0x0},
  len = 0xd8,
  data = 0xffff888107c4012c ""
}

crash> nf_conn_priv -x ffff888107c40168      
struct nf_conn_priv {
  info = {{
//省略部分
      stats = {0xffff88817fca8e00, 0x0}		//是priv_stats结构指针
    }
```
也就是说 0xffff88817fca8e00 既指向 nf_ct_ext，又指向 priv_stats。

而造成nf_ct_ext.offset[0,1] 异常的原因，应该就是priv_stats统计更新将其写坏了。
这部分正好对应priv_stats的一个统计字段。会将其当作 统计进行inc/dec操作。

比较可能的推理就是， 正常时 这里的值 0x01180000, 当dec 3次后，会变成0x1117fffd， 从而和我们看到的一致。
offset = {0xfffd, 0x117,...}


我们再看回CPU0的异常栈。nf_ct_ext中增加了dcache缓存相关dst_entry，会不会CPU0的 dst和CPU1的privcache有关系呢？

```c
crash> nf_ct_privcache -x 0xffff88817fca8e30
struct nf_ct_privcache {
  rt = {0xffff888150ba8575, 0xffff88823154a600}
}
```
0xffff888150ba8575		-- CPU1 dcache
0xffff888150ba6b0e		-- CPU0 skb->_skb_refdst 
看起来不一样，但是有点接近。推断是priv_stats操作更新统计时，将对应位置的rtable指针改掉了。

至此问题开始清晰， 同一块内存，被两个结构使用，一个结构对内存的操作，导致另一个结构访问到错误的信息。
为什么会被两个结构使用？推断A使用的内存被异常释放，然后被重新申请后给B使用，导致了A，B共用同一块内存。

那么是谁先异常释放的？ 因为priv_stats 申请会清空内存，但内存中 nf_ct_ext offset等字段相对完整，可以确认是 priv_stats先异常释放，然后再被nf_ct_ext申请使用。

于是开始分析priv_stats的异常释放原因。
## coredump 2 号
不多久，同一个客户又上报说有新的crash。PANIC: "kernel BUG at mm/slub.c:3942!" 触发内核 BUG_ON。
### 异常栈分析

```c
 #7 [ffffc9001c8a3ce0] invalid_op at ffffffff80c00ebb
    [exception RIP: kfree+335]
    RIP: ffffffff8037982f  RSP: ffffc9001c8a3d98  RFLAGS: 00010246
    RAX: ffffea0004447ca0  RBX: ffff888247368400  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: ffffc90000c3b600  RDI: ffffea0000000000
    RBP: ffffc9001c8a3db0   R8: 0000000000000000   R9: 0000000000003377
    R10: ffffea00091cda00  R11: ffff8882388bf550  R12: 0000000000000000
    R13: ffffffff80990e97  R14: ffff888247368400  R15: ffff888247368400
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #8 [ffffc9001c8a3db8] priv_stats_destroy at ffffffff80990e97
 #9 [ffffc9001c8a3df8] nf_conntrack_destroy at ffffffff80852fb2


crash> kmem ffff888247368400	//不在kmalloc-512的slab中，且不是复合page
ZONE  NAME        SIZE    FREE      MEM_MAP       START_PADDR  START_MAPNR
  2   Normal    1568768  674581  ffffea0004000000   100000000     1048575  
AREA    SIZE  FREE_AREA_STRUCT
  1       8k  ffffffff814f9948  
ffffea00091cda00  (ffff888247368400 is 1st of 2 pages)

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00091cda00 247368000                0 ffff888247368400  0 8000000000000000

```
对应代码

```c
void kfree(const void *x)
{
        struct page *page;
        void *object = (void *)x;

        trace_kfree(_RET_IP_, x);

        if (unlikely(ZERO_OR_NULL_PTR(x)))
                return;

        page = virt_to_head_page(x);
        if (unlikely(!PageSlab(page))) {
                BUG_ON(!PageCompound(page));	//
                kfree_hook(x);
                __free_pages(page, compound_order(page));
                return;
        }
        slab_free(page->slab_cache, page, object, NULL, 1, _RET_IP_);
}

```

基本上是这个地址已经不在kmalloc-512的slab中，且不是复合page，触发内核的BUG_ON.
表明这个地址已经被释放过，这次再释放触发了crash。

又是priv_stats的释放问题，这次是重复释放。
基本敲定了priv_stats的相关代码存在问题。

## 结合 coredump 1 和 2 
两次crash都和priv_stats异常释放相关。于是查阅priv_stats相关代码，发现priv_stats有自己的 refcnt，和inc/dec封装，当refcnt dec到0时对其释放。在各个位置都有上锁。没有发现明显问题。
测试环境也无法复现。暂时搁置去忙其他问题了。
# 二、 再起波澜
几个月后，另一个客户也遇到了系统crash，这次的现象更加怪异。

## coredump 3号 异常栈分析
问题现象 PANIC: "BUG: unable to handle kernel paging request at ffffffff00001848"

```c
 #8 [ffff88847ec038b0] page_fault at ffffffff80c01315
    [exception RIP: nf_reinject+55]
    RIP: ffffffff808629e7  RSP: ffff88847ec03968  RFLAGS: 00010282
    RAX: ffffffff00001848  RBX: ffff88846bc93000  RCX: 0000000000000001
    RDX: 0000000000000309  RSI: 0000000000000001  RDI: ffff88842621dc80
    RBP: ffff88847ec039a0   R8: ffffffff807f53ce   R9: ffffffff807f54c2
    R10: ffffea0010988740  R11: 0000000000000004  R12: ffff88842621dc80
    R13: 0000000100000000  R14: 0000000000000034  R15: ffff8884347f63c0
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #9 [ffff88847ec039a8] imq_dev_xmit at ffffffff80669000
#10 [ffff88847ec039d0] dev_hard_start_xmit at ffffffff8080b94c
#11 [ffff88847ec03a28] __imq_nf_queue at ffffffff80668916
#12 [ffff88847ec03a88] imq_nf_queue at ffffffff806692df
#13 [ffff88847ec03ae0] nf_queue at ffffffff8086290a

```
查看反汇编

```c
crash> dis -r nf_reinject+55
0xffffffff808629b0 <nf_reinject>:       nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff808629b5 <nf_reinject+5>:     push   %rbp
0xffffffff808629b6 <nf_reinject+6>:     mov    %rsp,%rbp
0xffffffff808629b9 <nf_reinject+9>:     push   %r15
0xffffffff808629bb <nf_reinject+11>:    push   %r14
0xffffffff808629bd <nf_reinject+13>:    push   %r13
0xffffffff808629bf <nf_reinject+15>:    push   %r12
0xffffffff808629c1 <nf_reinject+17>:    mov    %rdi,%r12	//nf_queue_entry
0xffffffff808629c4 <nf_reinject+20>:    push   %rbx
0xffffffff808629c5 <nf_reinject+21>:    sub    $0x10,%rsp
0xffffffff808629c9 <nf_reinject+25>:    mov    0x20(%rdi),%eax	//entry->state.hook
0xffffffff808629cc <nf_reinject+28>:    movzbl 0x24(%rdi),%edx	//entry->state.pf
0xffffffff808629d0 <nf_reinject+32>:    mov    %esi,-0x34(%rbp)
0xffffffff808629d3 <nf_reinject+35>:    mov    0x10(%rdi),%r13
0xffffffff808629d7 <nf_reinject+39>:    lea    0x309(%rax,%rdx,8),%rdx	//
0xffffffff808629df <nf_reinject+47>:    mov    0x40(%rdi),%rax	//entry->state.net=0xffffffff00000000 异常
0xffffffff808629e3 <nf_reinject+51>:    lea    (%rax,%rdx,8),%rax
0xffffffff808629e7 <nf_reinject+55>:    mov    (%rax),%rax	//hooks = rcu_dereference(net->nf.hooks[pf][entry->state.hook]);

```
net地址异常导致后续获取hooks时crash。查看nf_queue_entry发现不止 net地址错误，很多字段都异常。

```c
crash> nf_queue_entry -x ffff88842621dc80
struct nf_queue_entry {
  list = {
    next = 0x0,
    prev = 0x0
  },
  skb = 0x100000000,
  id = 0x0,
  hook_index = 0x0,
  state = {
    hook = 0x0,
    pf = 0x0,
    in = 0x0,
    out = 0x0,
    sk = 0x0,
    net = 0xffffffff00000000,
    okfn = 0xffffffff80935f10 <ip_finish_output>
  },
  size = 0x68
}

crash> kmem ffff88842621dc80
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803500      128      52411    131072   4096     4k  kmalloc-128
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0010988740  ffff88842621d000     0     32         27     5
  FREE / [ALLOCATED]
   ffff88842621dc80  (cpu 0 cache)

```
进一步发现 内存不仅字段异常，而且已经被释放。
entry是nf_queue中申请的，

```c
crash> kmem 0xffff88842621dc80 
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803500      128      52411    131072   4096     4k  kmalloc-128
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0010988740  ffff88842621d000     0     32         27     5
  FREE / [ALLOCATED]
   ffff88842621dc80  (cpu 0 cache)

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0010988740 42621d000                0 ffff88842621d780  1 8000000000000100 slab
crash> kmem -S=0 ffff88846c803500  
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803500      128      52411    131072   4096     4k  kmalloc-128
CPU 0 KMEM_CACHE_CPU:
  ffff88847ec4bb50
crash> kmem_cache_cpu -x ffff88847ec4bb50
struct kmem_cache_cpu {
  freelist = 0xffff88842621dc80,	//释放给percpu的freelist
  tid = 0x14d37fd923,
  page = 0xffffea0010988740,
  partial = 0xffffea001090df00
}

```
通过以上可以确认改内存被CPU0 释放的，但nf_queue 这部分处理是在软中断中，也就是说这个释放操作只能是nf_queue到nf_reinject之间。而这部分代码没有存在释放nf_queue_entry的操作， 所以有其他操作将nf_queue_entry释放了。

扩大怀疑范围，将这个期间所有申请和释放操作都列出来分析。发现存在一个skb_priv_save/skb_priv_restore进行了内存申请和释放。
进一步 skb_priv_save/skb_priv_restore 操作的内存 和 nf_queue_entry一样都是在kmalloc-128 slab上申请内存。
![对比内存content](https://i-blog.csdnimg.cn/direct/359720a2948f49f495f742a8187fd0c5.png)
对比nf_queue_entry和skb_priv的内容，发现这块内存确实像是被skb_priv覆写了。也就是说0xffff88842621dc80这块内存，nf_queue_entry还在使用的过程中，又被skb_priv申请到，然后覆写了其中的一部分。

从逻辑上看，nf_queue_entry 申请后 没有释放的操作， 却又被skb_priv_save 申请到， 只能有一种可能：kmalloc-128的freelist存在指向自己的loop。
示例如下：

```bash
kmem_cache_cpu.freelist=0xffff88842621dc80
                           |
                           V
                      ————————————————————————————————————————
                      |0xffff88842621dc80                    |
                      ————————————————————————————————————————
```
1. freelist=0xffff88842621dc80 此时申请nf_queue_entry， 到内存 0xffff88842621dc80
2. 系统随后读取0xffff88842621dc80的前8个字节作为下一个内存， 但是前8个字节还是0xffff88842621dc80， 所以申请0xffff88842621dc80 之后，freelist还是指向0xffff88842621dc80。
3. skb_priv_save申请内存，从freelist中得到0xffff88842621dc80。
这样就形成了 同一块内存同时被两个结构操作。

造成这次异常的深层原因是freelist出现了loop，那么是什么问题造成的呢？？？
最简单的场景就是，释放内存X后，内存X被freelist指向，且X的前8字节改写成下一个free object的地址。此时如果修改内存X的前8个字节指向自己(use-after-free),就能造成这个问题。
无论 nf_queue_entry还是skb_priv的前8个字节都不是指针，所以即使有use-after-free也不会出现这种操作。

## 内存挖掘

搜索和相关的内存。

```c
crash> search 0xffff88842621dc80
ffff888402e9ac00: ffff88842621dc80 
ffff888402eb4200: ffff88842621dc80 
ffff888402eb5200: ffff88842621dc80 
ffff888402eb5600: ffff88842621dc80 
ffff888402edee00: ffff88842621dc80 

```
查看发现这些是nf_ct_ext结构，内存前8字节都是 ffff88842621dc80。
```c
crash> kmem ffff888402e9ac00
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803080      512      22636     26656   1666     8k  kmalloc-512
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea00100ba680  ffff888402e9a000     0     16         15     1
  FREE / [ALLOCATED]
  [ffff888402e9ac00]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00100ba680 402e9a000                0 ffff888402e9a600  1 8000000000008100 slab,head
crash> nf_ct_ext -x ffff888402e9ac00
struct nf_ct_ext {
  rcu = {
    next = 0xffff88842621dc80,
    func = 0x0
  },
  offset = {0x0, 0x1c0, 0x0, 0x30, 0x70, 0x50, 0x0, 0x0, 0x88, 0x180, 0x60, 0x98, 0x0},
  len = 0x1e0,
  data = 0xffff888402e9ac2c ""
}
crash> kmem ffff888402eb4200
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803080      512      22636     26656   1666     8k  kmalloc-512
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea00100bad00  ffff888402eb4000     0     16         15     1
  FREE / [ALLOCATED]
  [ffff888402eb4200]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00100bad00 402eb4000                0 ffff888402eb5a00  1 8000000000008100 slab,head
crash> nf_ct_ext -x ffff888402eb4200
struct nf_ct_ext {
  rcu = {
    next = 0xffff88842621dc80,
    func = 0x0
  },
  offset = {0x0, 0x1c0, 0x0, 0x30, 0x70, 0x50, 0x0, 0x0, 0x88, 0x180, 0x60, 0x98, 0x0},
  len = 0x1e0,
  data = 0xffff888402eb422c ""
}

```

为什么这两个nf_ct_ext的头8个字节都是 0xffff88842621dc80呢，这是因为 nf_ct_ext最初申请时是128字节，从kmalloc-128中申请，申请后内存头8个字节没有改写，所以还保留了next free object指针。后续realloc成512字节了，但是前面的内容copy过来了。所以前面8个字节保存的是之前的一个128字节的free object。

于是查看 ffff888402e9ac00 的内容

```c
crash> rd ffff888402e9ac00 64  
ffff888402e9ac00:  ffff88842621dc80 0000000000000000   ..!&............
ffff888402e9ac10:  0030000001c00000 0000000000500070   ......0.p.P.....
ffff888402e9ac20:  0098006001800088 0000000001e00000   ....`...........
ffff888402e9ac30:  0000000000000003 0000000000000084   ................
ffff888402e9ac40:  0000000000000001 0000000000000034   ........4.......
ffff888402e9ac50:  17db8b12cdb1e844 0000000000000000   D...............
ffff888402e9ac60:  ffff888422036600 ffff88842a8b1800   .f.".......*....
ffff888402e9ac70:  0000000000000000 0000ffffffff0000   ................
ffff888402e9ac80:  0000000000000000 0000000900000001   ................
ffff888402e9ac90:  00004040e0250cb4 00000000100d0003   ..%.@@..........
ffff888402e9aca0:  dead4ead00000000 00000000ffffffff   .....N..........
ffff888402e9acb0:  ffffffffffffffff 0000000000000000   ................
ffff888402e9acc0:  ffff88843d684200 0000000000fe0003   .Bh=............
ffff888402e9acd0:  dead4ead00000000 00000000ffffffff   .....N..........
ffff888402e9ace0:  ffffffffffffffff 0000000000000000   ................
ffff888402e9acf0:  0000000000000000 384e415700000000   ............WAN8
ffff888402e9ad00:  2e302e302e30315f 0000000000382f30   _10.0.0.0/8.....
ffff888402e9ad10:  0000000000000000 0000000000000000   ................
ffff888402e9ad20:  0000000000000000 0000000000000000   ................
ffff888402e9ad30:  0000000000000000 0000000000000000   ................
ffff888402e9ad40:  0000000000000000 0000000000000000   ................
ffff888402e9ad50:  0000000000000000 0000000000000000   ................
ffff888402e9ad60:  0000000000000000 0000000000000000   ................
ffff888402e9ad70:  0000000000000000 0000000000000000   ................
ffff888402e9ad80:  6c6c415f354e4157 0000000000000000   WAN8_All........
ffff888402e9ad90:  0000000000000000 0000000000000000   ................
ffff888402e9ada0:  0000000000000000 0000000000000000   ................
ffff888402e9adb0:  0000000000000000 0000000000000000   ................
ffff888402e9adc0:  0000000000000000 0000000000000000   ................
ffff888402e9add0:  0000000000000000 0000001300000000   ................
ffff888402e9ade0:  ffffea00108cb5c2 0000100000000000   ................
ffff888402e9adf0:  00000004232d7000 0000000000001000   .p-#............

```
对比

```c
crash> rd 0xffff88842621dc80 64
ffff88842621dc80:  0000000000000000 0000000000000000   ................
ffff88842621dc90:  0000000100000000 0000000000000000   ................
ffff88842621dca0:  0000000000000000 0000000000000000   ................
ffff88842621dcb0:  0000000000000000 0000000000000000   ................
ffff88842621dcc0:  ffffffff00000000 ffffffff80935f10   ........._......
ffff88842621dcd0:  0000000000000068 0000000000000000   h...............
ffff88842621dce0:  0000000000000000 0000000000000000   ................
ffff88842621dcf0:  0000000000000020 0000ffffffff0000    ...............
ffff88842621dd00:  0000000000000000 0000000900000001   ................
ffff88842621dd10:  00004040e0250cb4 00000000100d0001   ..%.@@..........
ffff88842621dd20:  dead4ead00000000 00000000ffffffff   .....N..........
ffff88842621dd30:  ffffffffffffffff 0000000000000000   ................
ffff88842621dd40:  0000000000000000 0000000000000000   ................
ffff88842621dd50:  dead4ead00000000 00000000ffffffff   .....N..........
ffff88842621dd60:  ffffffffffffffff 0000000000000000   ................
ffff88842621dd70:  0000000000000000 384e415700000000   ............WAN8
ffff88842621dd80:  3836312e3239315f 0036312f302e302e   _192.168.0.0/16.
ffff88842621dd90:  0000000000000000 0000000000000000   ................
ffff88842621dda0:  0000000000000000 0000000000000000   ................
ffff88842621ddb0:  0000000000000000 0000000000000000   ................
ffff88842621ddc0:  0000000000000000 0000000000000000   ................
ffff88842621ddd0:  0000000000000000 0000000000000000   ................
ffff88842621dde0:  0000000000000000 0000000000000000   ................
ffff88842621ddf0:  0000000000000000 0000000000000000   ................
ffff88842621de00:  ffff88842621d480 ffffffff803cbe00   ..!&......<.....
ffff88842621de10:  0000000000000000 ffff88842621de18   ..........!&....
ffff88842621de20:  ffff88842621de18 ffffffffffffffff   ..!&............
ffff88842621de30:  ffff888430a47540 000000010000000e   @u.0............
ffff88842621de40:  ffff88842621de40 ffff88842621de40   @.!&....@.!&....
ffff88842621de50:  ffff888423365800 ffff888430a47648   .X6#....Hv.0....
ffff88842621de60:  dead000000000200 0000000000000000   ................
ffff88842621de70:  0000000e00000019 0015001600000000   ................

```
按理说，ffff88842621dc80 ffff88842621dd00 ffff88842621dd80 ffff88842621de00 这几个128字节的object应该独立的。但是明显ffff88842621dd70 和 ffff88842621dd80 内容表示它们是连在一起使用的。像是 ffff88842621dc80 用作nf_ct_ext了，但nf_ct_ext 完整大小是512字节。 应该从kmalloc-512中申请， 不应该从kmallo-128申请。

此时问题驱使我去查看kmalloc-512的freelist

```c
crash> kmem -S=0 kmalloc-512
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803080      512      22636     26656   1666     8k  kmalloc-512
CPU 0 KMEM_CACHE_CPU:
  ffff88847ec4bbb0
CPU 0 SLAB:
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0010f5a100  ffff88843d684000     0     16         14     2
  FREE / [ALLOCATED]
  [ffff88843d684000]
crash> kmem_cache_cpu.freelist ffff88847ec4bbb0
  freelist = 0xffff88842621df00,	//注意这里
crash> rd 0xffff88842621df00 
ffff88842621df00:  ffff88842621dd80                    ..!&....		//注意这里
crash> rd ffff88842621dd80
ffff88842621dd80:  3836312e3239315f                    _192.168
crash> kmem 0xffff88842621df00 
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803500      128      52411    131072   4096     4k  kmalloc-128	//kmalloc-512的freelist指向kmalloc-128的object
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0010988740  ffff88842621d000     0     32         27     5
  FREE / [ALLOCATED]
  [ffff88842621df00]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0010988740 42621d000                0 ffff88842621d780  1 8000000000000100 slab
crash> kmem ffff88842621dd80  
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff88846c803500      128      52411    131072   4096     4k  kmalloc-128	//kmalloc-512的freelist指向kmalloc-128的object
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0010988740  ffff88842621d000     0     32         27     5
  FREE / [ALLOCATED]
  [ffff88842621dd80]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0010988740 42621d000                0 ffff88842621d780  1 8000000000000100 slab

```
至此你看到了可能一辈子都难以看到的。kmalloc-512的freelist指向kmalloc-128的object，闻所未闻啊。

## kmalloc-128 和 kmalloc-512 的羁绊
所有人到这里都会问一句，为什么kmalloc-512会使用kmalloc-128的object？？？
结合之前的coredump，经过仔细思考，可能的逻辑如下：
1. priv_stats异常释放，freelist指向内存X，内存X头8个字节指向下个free object B。
2. nf_ct_ext_add调用realloc扩展到512大小，申请到内存X。此时freelist指向下一个free object B。
3. [异常场景]priv_stats再次释放内存X，导致 freelist指向内存X， 内存X头8个字节指向下个free object B。
4. nf_ct_ext_add将内容从旧内存拷贝到内存X中，改写内存X头8个字节为ffff88842621dc80。此时kmalloc-512的freelist指向 内存X， 内存X的头8个字节指向 ffff88842621dc80。
5. 再次申请kmalloc-512 得到内存X(多个结构都在使用内存X，此时暂且不讨论)。再次再次申请kmalloc-512 得到内存ffff88842621dc80。此时freelist指向ffff88842621dc80内存头8个字节的另一个128字节ojbect。而同一时刻kmalloc-128的freelist中也包含ffff88842621dc80。
6. 虽然ffff88842621dc80是通过kmalloc-512申请得到的。但kfree释放ffff88842621dc80会将其归还到kmalloc-128的freelist，此时kmalloc-128的freelist就有了两个ffff88842621dc80 object


问题就这样出现了。虽然很诡异，但是就是出现了。根源仍然是存在priv_stats的异常释放。

# 三、 事不过三
为了定位问题，除了KASAN版本，也出了相关debug image，但客户加载后久久都不再出现。价值还有其他事情，就又搁置了一段时间，但是心里老是再琢磨这个问题。
前不久，另一个客户上报crash问题，初步查看发现也是priv_stats相关，下意识就觉得和之前出现的问题有关。仔细一看果不其然。
##  coredump 4号 异常栈
crash原因 PANIC: "BUG: unable to handle kernel paging request at 00000000544c541d"

```c
 #8 [ffff88847ec03980] page_fault at ffffffff80c01315
    [exception RIP: priv_stats_find_by_gateway+78]
    RIP: ffffffff8099e1be  RSP: ffff88847ec03a38  RFLAGS: 00010212
    RAX: 00000000544c53b9  RBX: 0000000000000056  RCX: 0000000000000000
    RDX: ffff8884610e0000  RSI: 00000000d265f34e  RDI: 00000000ed0f3f7e
    RBP: ffff88847ec03a50   R8: 0000000000000002   R9: 00000000d65e76b9
    R10: 0000000000000000  R11: 0000000000000000  R12: 00000000d65e76b9
    R13: ffffffff814da980  R14: 00000000d65e76b9  R15: ffff8884610e0000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #9 [ffff88847ec03a58] priv_stats_set_exgress_gateway at ffffffff809a0555
#10 [ffff88847ec03aa8] priv_stats_exgress_update at ffffffff809a0fd0
#11 [ffff88847ec03ae8] ip_finish_output2 at ffffffff80934232
#12 [ffff88847ec03b38] ip_finish_output at ffffffff809364d5
#13 [ffff88847ec03b70] ip_output at ffffffff80937348
#14 [ffff88847ec03be8] ip_forward_finish at ffffffff80931cba
#15 [ffff88847ec03c10] ip_forward at ffffffff80932351
#16 [ffff88847ec03c70] ip_rcv_finish at ffffffff8092e78a
#17 [ffff88847ec03ca0] ip_rcv at ffffffff8093010a
#18 [ffff88847ec03d28] __netif_receive_skb_core at ffffffff8080c58a
#19 [ffff88847ec03db8] __netif_receive_skb at ffffffff8080ccb8

```
通过priv_hash list，得到 ffff8884263feb88, 从而得到priv_stats的指针 ffff8884263fea00。

问题出现在访问priv_stats.list.next时。

```c
crash> priv_stats.list -x ffff8884263fea00
  list = {
    next = 0x544c5541,	//错误指针，导致crash
    pprev = 0x0
  },

crash> priv_stats.list -xo                
struct priv_stats {
  [0x188] struct hlist_node list;
}

```
我们再用nf_ct_ext来解读 ffff8884263fea00

```c
crash> nf_ct_ext -x ffff8884263fea00
struct nf_ct_ext {
  rcu = {
    next = 0xffff888422fade80,
    func = 0x0
  },
  offset = {0x0, 0x1c0, 0x0, 0x30, 0x70, 0x50, 0x0, 0x0, 0x393a, 0x180, 0x60, 0x98, 0x0},
  len = 0x0,
  data = 0xffff8884263fea2c ""
}

crash> rd ffff8884263feb80 8 
ffff8884263feb80:  4645445f33505349 00000000544c5541   ISP3_DEFAULT....
ffff8884263feb90:  0000000000000000 0000000000000000   ................
ffff8884263feba0:  0000000000000000 0000000000000000   ................
ffff8884263febb0:  0000000000000000 0000000000000000   ................
```

nf_ct_ext 在这个位置写入了字符串，被priv_stats解读成hash list，访问出错。

依然是 priv_stats异常释放后的user-after-free导致的问题。因为有了前几次的经验，所以这次分析的非常快。

## 水落石出
这次决定静下来，好好的分析下priv_stats相关代码，必须找到原因。
先是梳理了几个关键的 设置和清除点，以及对应的refcnt get/put 。第一遍下来，发现代码处理相对完整，set的地方有refcnt get，clear的地方有refcnt put。
于是再看第二遍，发现有一个清除的操作不太对劲。又将对应设置部分代码拿来比较，终于看到端倪。
在nf_conn_priv中存在4个priv_stats[0,1,2,3]， 每个指针赋值时都会将对应priv_stats的递增refcnt。但是在释放时，却要求每个priv_stats都和另外3个不同才会 递减refcnt。示例如下：

```c
        if (pst1) {
                refcnt_put(pst1);
        }

        if (pst2 && pst2 != pst1) {
                refcnt_put(pst2);
        }

        if (pst3 && pst3 != pst1 && pst3 != pst2) {
                refcnt_put(pst3);
        }

        if (pst4 && pst4 != pst1 && pst4 != pst2 && pst4 != pst3) {
                refcnt_put(pst4);
        }
```

这会导致 refcnt leak。之前一直想那里多 递减了refcnt， 但实际问题是 少递减了 refcnt。结果就是refcnt会越来越多，直到32整型溢出反转。当反转到0后，此时一次简单的 inc/dec操作，就会触发priv_stats的释放。从而导致use-after-free，常见的就是priv_stats和nf_ct_ext共用内存。因为32整型溢出直到变回0 需要很长时间，所以这个问题很难复现，客户一旦重启后又久久不再出现。

# 四、 后记
因为同时有很多问题处理，所以没有集中去分析这个问题，断断续续的，导致没有能很快解决掉。每次重新再分析时都要再看一遍异常栈将思路找回。好在最后找到原因解决了，了了近期的一块心病。

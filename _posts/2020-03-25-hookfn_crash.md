---
title: "[crash analysis]NULL pointer dereference caused by HOOK okfn"
date: 2020-03-25
---

设备启用业务模块，同时又创建了桥口转发docker业务。在调试过程中，出现crash。

```powershell
[  794.020484] BUG: unable to handle kernel
[  794.022499] NULL pointer dereference at 0000000000000006
[  794.024455] IP: [<ffffffffa0330ee3>] br_nf_pre_routing_finish+0x53/0x450 [br_netfilter]
...	//省略无关
[  794.101362]  [<ffffffff815aee4c>] ? __ip_route_output_key_hash+0x32c/0x930
[  794.103338]  [<ffffffffa0750bbc>] Test_LocalDeliver+0xfc/0x260 [testmod]
[  794.105446]  [<ffffffffa06b8663>] _Test_DoActions+0x483/0x1060 [testmod]
[  794.107566]  [<ffffffffa033b33d>] ? nf_conntrack_free+0x4d/0x60 [nf_conntrack]
[  794.109725]  [<ffffffffa033c9b0>] ? destroy_conntrack+0xa0/0x100 [nf_conntrack]
[  794.111876]  [<ffffffffa0093da3>] ? start_xmit+0x253/0x4f0 [virtio_net]
[  794.113901]  [<ffffffffa06becdd>] ? LW_FlowTableLookup+0x1ad/0x950 [testmod]
[  794.115942]  [<ffffffffa06ba1db>] Test_Process+0x85b/0xea0 [testmod]
```
查看异常栈，crash在br_nf_pre_routing_finish+0x53/0x450位置，查看反汇编。

```powershell
crash> dis -r br_nf_pre_routing_finish+0x53
0xffffffffa0330e90 <br_nf_pre_routing_finish>:  nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffffa0330e95 <br_nf_pre_routing_finish+5>:        push   %rbp
0xffffffffa0330e96 <br_nf_pre_routing_finish+6>:        mov    %rsp,%rbp
0xffffffffa0330e99 <br_nf_pre_routing_finish+9>:        push   %r15
0xffffffffa0330e9b <br_nf_pre_routing_finish+11>:       push   %r14
0xffffffffa0330e9d <br_nf_pre_routing_finish+13>:       mov    %rdi,%r14
0xffffffffa0330ea0 <br_nf_pre_routing_finish+16>:       push   %r13
0xffffffffa0330ea2 <br_nf_pre_routing_finish+18>:       push   %r12
0xffffffffa0330ea4 <br_nf_pre_routing_finish+20>:       push   %rbx
0xffffffffa0330ea5 <br_nf_pre_routing_finish+21>:       mov    %rsi,%rbx
0xffffffffa0330ea8 <br_nf_pre_routing_finish+24>:       sub    $0x90,%rsp
0xffffffffa0330eaf <br_nf_pre_routing_finish+31>:       mov    0x20(%rsi),%r13
0xffffffffa0330eb3 <br_nf_pre_routing_finish+35>:       mov    0x90(%rsi),%r12	//r12 = [rsi + 0x90]
0xffffffffa0330eba <br_nf_pre_routing_finish+42>:       mov    %gs:0x28,%rax
0xffffffffa0330ec3 <br_nf_pre_routing_finish+51>:       mov    %rax,-0x30(%rbp)
0xffffffffa0330ec7 <br_nf_pre_routing_finish+55>:       xor    %eax,%eax
0xffffffffa0330ec9 <br_nf_pre_routing_finish+57>:       movzwl 0x3a(%rsi),%eax
0xffffffffa0330ecd <br_nf_pre_routing_finish+61>:       movzwl 0xc2(%rsi),%r15d
0xffffffffa0330ed5 <br_nf_pre_routing_finish+69>:       mov    0xe0(%rsi),%rcx
0xffffffffa0330edc <br_nf_pre_routing_finish+76>:       mov    0x3e8(%r13),%r9
0xffffffffa0330ee3 <br_nf_pre_routing_finish+83>:       mov    %ax,0x6(%r12)	//问题位置
```

查看问题指令可知 r12寄存器为0，对r12 指向地址偏移6位置的访问导致异常地址访问。r12是skb（
rsi）偏移0x90得到。可知 r12是skb->nf_bridge。也就是说skb->nf_bridge 为NULL导致了问题发生。
查看代码 br_nf_pre_routing_finish 是注册在bridge hook的 NF_BR_PRE_ROUTING链上的br_nf_pre_routing中的处理。从代码中看nf_bridge不应该为NULL的。

```c
static unsigned int br_nf_pre_routing(const struct nf_hook_ops *ops,
          struct sk_buff *skb,
          const struct net_device *in,
          const struct net_device *out,
          const struct nf_hook_state *state)
{
...//省略部分
 if (!nf_bridge_alloc(skb))
  return NF_DROP;
  ...//省略部分
   nf_bridge = nf_bridge_info_get(skb);
   nf_bridge->ipv4_daddr = ip_hdr(skb)->daddr;
   
  skb->protocol = htons(ETH_P_IP);
  NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, state->sk, skb,
    skb->dev, NULL, br_nf_pre_routing_finish);
  return NF_STOLEN;
}
```

随后将疑点转向调用栈，发现业务模块 Test_LocalDeliver-->br_nf_pre_routing_finish。查看Test_LocalDeliver函数，发现是通过调用保存的L3RxOkfn来将数据包方向本机。
这里明明保存的三层的okfn，怎么调用到二层hook链上的okfn了？
(这里简单介绍下L3RxOkfn，我们在三层NF_INET_PRE_ROUTING注册了hook，将包收上来后，将对应的okfn保存到L3RxOkfn中，随后将包stolen，放到业务模块的链上。在task调度中，业务模块将包从链上取出进行处理后，发向本机的包，通过L3RxOkfn来继续处理)。

正常情况下我们在三层的hook抓包后，L3RxOkfn应该是ip_rcv_finish

```c
ip_rcv 函数的末尾
 return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, NULL, skb,
         dev, NULL,
         ip_rcv_finish);

br_nf_pre_routing的末尾
 NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, state->sk, skb,
  skb->dev, NULL,
  br_nf_pre_routing_finish);
```

分析到这里，问题基本清晰了。

1. 某个数据包在bridge接口的 PREROUTING中处理，进入了br_nf_pre_routing，然后进入了 NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, state->sk, skb, skb->dev, NULL,br_nf_pre_routing_finish)。
2. 进入NF_INET_PREROUTING后，被业务模块三层hook钩住， 会更新 L3RxOkfn = br_nf_prerouting_finish。
3. 此后tasklet调度到业务模块，处理完送到本机的包(这是普通的三层包，没有nfbridge)，调用gsL3RxOkfn 出现crash。

这个问题是概率出现，因为在tasklet调度到业务模块之前，有新包在三层PREROUTINGhook处理，会将L3RxOkfn 重新更新成ip_rcv_finish，这样后续处理就没问题了。

如果不想要二层birdge桥口hook处理中调用三层netfilter call tables。可以echo 0 > /proc/sys/net/bridge/bridge-nf-call-iptables 关闭。.

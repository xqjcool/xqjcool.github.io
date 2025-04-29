---
title: "SSH login failure caused by an uninitialized structure variable."
date: 2025-04-29
---

# 结构变量未初始化导致的ssh登录失败

## 问题现象
QA上报说测试设备无法ssh连接，web页面是可以登录的。也就是说22端口连接异常，https的443端口没有问题。

## 初步观察
尝试ssh连接设备，被拒。

```bash
$ ssh admin@10.21.43.90 
ssh: connect to host 10.21.43.90 port 22: Connection refused
```

通过web页面管理进入设备shell，tcpdump观察报文交互。

```bash
/var/log# tcpdump -i eth0 tcp port 22
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:57:10.025407 IP 172.18.29.154.54090 > 10.21.43.90.22: Flags [S], seq 4105399158, win 64240, options [mss 1398,sackOK,TS val 456890810 ecr 0,nop,wscale 11], length 0
17:57:10.025444 IP 10.21.43.90.22 > 172.18.29.154.54090: Flags [R.], seq 0, ack 4105399159, win 0, length 0
```

可以看到tcp连接请求被rst了。那么是谁拒绝的sshd还是内核？因为什么原因呢？

对sshd进程进行trace，观察是否有accept处理。

```bash
/var/log# strace -p 4966
strace: Process 4966 attached
ppoll([{fd=3, events=POLLIN}, {fd=4, events=POLLIN}], 2, NULL, [], 8
```

ssh请求期间 进程处于poll等待，没有accept处理。也就是说报文并没有送给进程处理，那就是在内核中就被rst了。

是iptables规则导致的么？查看没有发现异常规则。

```bash
/# iptables -A INPUT -s 172.18.29.154
/# iptables -nvL
Chain INPUT (policy ACCEPT 6918K packets, 561M bytes)
 pkts bytes target     prot opt in     out     source               destination    
25  1300            0    --  *      *       172.18.29.154       0.0.0.0/0
```

添加监视，发现报文进入了filter的INPUT链。

## 继续深入

想要进一步了解更多信息，就需要上调试工具了。
我用[KernelDebugger](https://github.com/xqjcool/KernelDebugger)编写了个ko，去hook内核的tcp_v4_rcv函数，尝试观察目的端口为22的报文。

```c
HOOK_DEFINE(tcp_v4_rcv, int, struct sk_buff *skb)
{
        int ret;
        const struct iphdr *iph = ip_hdr(skb);
        const struct tcphdr *th;
 
        if (pskb_may_pull(skb, sizeof(struct tcphdr))) {
                th = (const struct tcphdr *)((char *)iph + iph->ihl * 4);
                if (ntohs(th->dest) == 22) {
                        pr_info("[%s:%d]: %pI4:%u --> %pI4:%u \n", __func__, __LINE__, &iph->saddr, ntohs(th->source), &iph->daddr, ntohs(th->dest));
                }
 
        }
        ret = CALL_ORIG_FUNCION(tcp_v4_rcv, skb);
        return ret;
}
```

发现没有打印，当时认为报文在tcp_v4_rcv之前就被rst了。 --这里犯了个小错误，按下不表。
接下来怀疑是`NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,net, NULL, skb, skb->dev, NULL,ip_local_deliver_finish)`处理过程中，某个hook将报文rst了。

于是写了个ko，去hook内核nf_hook_slow函数，结果竟然没有打印。？？？？？

## 编译优化搞的鬼
因为这个NF_HOOK是在ip_local_deliver中调用的，我用objdump查看vmlinux中ip_local_deliver的反汇编，里面没有call nf_hook_slow的指令。它直接把nf_hook_slow的代码嵌入到ip_local_deliver的函数中。

正常来说 nf_hook_slow不是静态函数，不应该内嵌进去的。但是一些编译优化会为了效率，将一些函数嵌入。效率是提升了，但却给调试带来了麻烦。 :-(

为了调试nf_hook_slow, 还得 hook ip_local_deliver函数。

调试ko做了以下几件事

- hook ip_local_deliver_finish (这个函数没有导出，不能直接调用，所以需要hook出来，后面ip_local_deliver的重构需要)
- hook nf_hook_slow (用来调试tcp请求的动向)
- hook ip_local_deliver (重写该函数，主要为了触发nf_hook_slow的显式调用，从而nf_hook_slow的调试能够生效)

打印发现nf_hook_slow 链上有5个hook， 在遍历到 s=2时， 没有再打印了。但verdict是ACCEPT也就是应该继续下一个hook的。增加调试信息，发现s=2对应 xxxx_filter模块。

```bash
s:2 hook:init_module+0xe60/0x1170 [xxxx_filter]
```

查看代码，发现这个模块在某些条件下会修改报文目的端口！瞬间明白了为什么 tcp_v4_rcv 没有打印了。 因为我的log条件是目的端口等于22，一旦目的端口值改变，自然就不会触发打印了。

于是修改调试代码，打印整个4元组。发现在 xxxx_filter模块hook处理后，目的端口改成了9

```bash
Apr 25 15:16:50 kern.info kernel: [160127.921711] [hook_nf_hook_slow:202]: 172.18.29.154:56684 --> 10.21.43.90:22 hook:1 pf:2 num:6
Apr 25 15:16:50 kern.info kernel: [160127.921715] [hook_nf_hook_slow:213]: 172.18.29.154:56684 --> 10.21.43.90:22 s:0 hook:parse_packet+0x210/0x360 [xxxx_filter]
Apr 25 15:16:50 kern.info kernel: [160127.921725] [hook_nf_hook_slow:213]: 172.18.29.154:56684 --> 10.21.43.90:22 s:1 hook:ipt_register_table+0x470/0x470
Apr 25 15:16:50 kern.info kernel: [160127.921731] [hook_nf_hook_slow:213]: 172.18.29.154:56684 --> 10.21.43.90:9 s:2 hook:init_module+0xe60/0x1170 [xxxx_filter]
```

## 查阅code

查看模块代码，找到修改目的端口的相关逻辑。发现当目的端口等于 custom_port时,会修改目的端口为 9

```c
#define APACHE_CUSTOM_PORT 9
if (ntohs(th->dest) == cur->custom_port) {
	th->dest = htons(APACHE_CUSTOM_PORT);
}
```

也就是说有个acl的custom_port = 22， 但是查看规则配置中，没有配置custom_port = 22的。

## crash工具调试

使用crash vmlinux实时调试当前内核，加载带符号的xxxx_filter.ko.debug。
查看相关acl规则

```bash
crash> dis xxxx_nat_get_port     
0xffffffffa04288f0 <xxxx_nat_get_port>:    nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffffa04288f5 <xxxx_nat_get_port+5>:  mov    %edi,%eax
0xffffffffa04288f7 <xxxx_nat_get_port+7>:  bswap  %eax
0xffffffffa04288f9 <xxxx_nat_get_port+9>:  mov    $0x824a4e61,%ecx
0xffffffffa04288fe <xxxx_nat_get_port+14>: imul   %rax,%rcx
0xffffffffa0428902 <xxxx_nat_get_port+18>: shr    $0x28,%rcx
0xffffffffa0428906 <xxxx_nat_get_port+22>: imul   $0x1f7,%ecx,%ecx
0xffffffffa042890c <xxxx_nat_get_port+28>: sub    %ecx,%eax
0xffffffffa042890e <xxxx_nat_get_port+30>: mov    -0x5fbd5000(,%rax,8),%rcx //hlist_head
0xffffffffa0428916 <xxxx_nat_get_port+38>: test   %rcx,%rcx

```

通过反汇编找到hlist_head数组的地址 0xffffffffa042b000

```bash
crash> rd 0xffffffffa042b000 512
ffffffffa042b000:  0000000000000000 0000000000000000   ................
//省略空白
ffffffffa042b900:  ffff888884161290 0000000000000000   ................
//省略空白
ffffffffa042bc10:  ffff888884161710 0000000000000000   ................
//省略空白
```

查看对应结构

```bash
crash> xxxx_nat_node_t -x ffff888884161200
struct xxxx_nat_node_t {
  ip = 0x5a2b150a,
  custom_port = 0x0,
  nproto = 0x4,
  proto = {
      protocol = 0x6,
      port = 0x1bb,
      action = 0x1,
      type = 0x4
    }, {
      protocol = 0x1,
      port = 0x0,
      action = 0x1,
      type = 0x1
    }, {
      protocol = 0x6,
      port = 0x16,
      action = 0x1,
      type = 0x2
    }, {
      protocol = 0x6,
      port = 0x50,
      action = 0x1,
      type = 0x3
    }, 
	//省略无关
}

crash> xxxx_nat_node_t -x FFFF888884161680
struct xxx_nat_node_t {
  ip = 0x5a2b150a,
  custom_port = 0x16,	//端口22 问题出在这里
  nproto = 0x4,
  proto = {
      protocol = 0x6,
      port = 0x1bb,
      action = 0x1,
      type = 0x4	//4
    }, {
      protocol = 0x1,
      port = 0x0,
      action = 0x1,
      type = 0x1	//1
    }, {
      protocol = 0x6,
      port = 0x16,
      action = 0x1,
      type = 0x2	//2
    }, {
      protocol = 0x6,
      port = 0x50,
      action = 0x1,
      type = 0x3	//3
    },
//省略无关
}
```

确实有node的 custom_port = 0x16,导致目的端口被改为9。

查看代码，只有当type 等于 XXXX_TYPE_CUSTOM = 19 时，才会设置 custom_port. 而proto中没有type为19的。也就是没有触发custom_port的赋值。

```c
new = kmalloc(sizeof(xxxx_nat_node_t), GFP_ATOMIC);
//省略无关
if (inf->proto[i].type == XXXX_TYPE_CUSTOM) {
	new->custom_port = inf->proto[i].port;
}
```

那么custom_port的值就是申请时内存中的残留值，这是个不可预测的。只是刚好这次是 22。导致了本次的问题。


解决的方法很简单，将 kmalloc ==> kzalloc就行了。

## 后记
埋下一个bug很容易，定位一个bug却是耗尽精力。
同志们，变量和结构的初始化一定要切记啊！！！

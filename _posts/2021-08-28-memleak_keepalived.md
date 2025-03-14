---
title: "[Memory Leak] Debugging and Analysis of keepalived Process Memory Leak."
date: 2021-08-28
---

# 缘起
公司部分项目在部署上通过keepalived实现了了双机热备，通过荣誉和接管，实现高可用性。

最近其中一个项目出现了内存耗尽问题，使用<内存问题快速定位工具>提示是keepalived进程的rarp socket收报队列挤压了太多数据包导致。
![image](https://github.com/user-attachments/assets/e822201e-6623-49ed-8f99-0072831d85ac)

# 分析
keepalived我们用的是v2.0.10版本，最新版本已经到v2.2.4了。于是查找keepalived的提交记录，看看问题是不是在新版本解决了。找到一个类似问题。
https://github.com/acassen/keepalived/issues/839

```
root@node02 ~ # ss -a | awk '{ if ($3 > 0 or $4 > 0) {print $0} }'
Netid  State      Recv-Q Send-Q   Local Address:Port       Peer Address:Port   
raw    UNCONN     37749184 0                    *:vrrp                  *:*    
```
该问题是18年4月份左右提出的，7月份就解决了。而v2.0.10版本是18年11月份发布的，所以应该已经包含问题的修复。

最新的v2.2.4和v2.0.10之间看起来没有该问题的修复。为了验证推测，编译了v2.2.4的keepalived，替换后进行问题复现测试。向设备发送大量rarp广播报文，然后ss观察rarp socket是否持续上涨。

结果是v2.2.4依然存在相同问题。

```
[root@localhost ~]# ss -pf link
Netid Recv-Q Send-Q                                                            Local Address:Port                                                                             Peer Address:Port                
p_raw 204800896 0                                                                          rarp:*                                                                                            *                      users:(("keepalived",pid=2530,fd=12))
[root@localhost ~]# 
```
通过升级版本来解决问题的方式行不通了。

# 提出问题
于是在github上向keepalived提了一个issue，https://github.com/acassen/keepalived/issues/1973。
描述了问题的现象，当时的keepalived版本和配置，问题的复现方法。

等待的过程中，我想何不研究下源码，因为这个问题基本上就是某个socket创建后只用来发送，没有对接收到的队列进行处理。收到报文越来越多，就导致的收包队列积压，造成内存消耗。

查看源码，发现vrrp_arg.c中创建了一个socket用来发送gratuitous ARP报文。

```
void gratuitous_arp_init(void)
{
	if (garp_buffer)
		return;
 
	/* Create the socket descriptor */
	garp_fd = socket(PF_PACKET, SOCK_RAW | SOCK_CLOEXEC | SOCK_NONBLOCK, htons(ETH_P_RARP));
 
	if (garp_fd >= 0) {
		if (__test_bit(LOG_DETAIL_BIT, &debug))
			log_message(LOG_INFO, "Registering gratuitous ARP shared channel");
	} else {
		log_message(LOG_INFO, "Error %d while registering gratuitous ARP shared channel", errno);
		return;
	}
 
	/* Initalize shared buffer */
	garp_buffer = PTR_CAST(char, MALLOC(GARP_BUFFER_SIZE));
}
```

但是这个garp_fd并没有处理receive queue，如果有rarp报文过来，就会永久queue在自己的接收队列，造成内存泄漏。

PS：细心的网友会产生疑问，发送ARP报文的socket，为何使用ETH_P_RARP作为协议。这里我当时也有点懵，猜想当时是作者无心造成的笔误，好在没有影响正常功能。

# 修复1
既然找到了原因，那就修复吧。经过调研，认为设置SO_RCVBUF为0可以阻止socket收包队列积压数据报文。
PS：查看kernel源码，设置SO_RCVBUF最小值不是0，而是。所以队列会有少量字节的报文囤积，不过因为量太小（2304bytes），对内存的影响基本可以忽略。

```
#define TCP_SKB_MIN_TRUESIZE	(2048 + SKB_DATA_ALIGN(sizeof(struct sk_buff)))
 
#define SOCK_MIN_SNDBUF		(TCP_SKB_MIN_TRUESIZE * 2)
#define SOCK_MIN_RCVBUF		 TCP_SKB_MIN_TRUESIZE
```

编译debug版本验证，发现收包队列在积压少量字节后(<=2304)，不再增长。验证有效。
于是向keepalived提交了pull request.

# 修复2
keepalived的维护者pqarmitage看到pull request后，本着精益求精的追求，提出了是否能够尝试shutdown(fd, SHUT_RD)的方式来更完美的关闭socket接收功能。
我修改后编译debug版本，进行验证，发现shutdown函数返回错误。

```
Aug 27 12:48:08 cpe Keepalived_vrrp[94304]: Assigned address 169.254.201.3 for interface eth5
Aug 27 12:48:08 cpe Keepalived_vrrp[94304]: Registering gratuitous ARP shared channel
Aug 27 12:48:08 cpe Keepalived_vrrp[94304]: shutdown SHUT_RD for socket 12 failed (95) - Operation not supported
```

查看kernel源码发现AF_PACKET socket不支持shutdown操作:

```
static const struct proto_ops packet_ops = {
	.family =	PF_PACKET,
	.owner =	THIS_MODULE,
	.release =	packet_release,
	.bind =		packet_bind,
	.connect =	sock_no_connect,
	.socketpair =	sock_no_socketpair,
	.accept =	sock_no_accept,
	.getname =	packet_getname,
	.poll =		packet_poll,
	.ioctl =	packet_ioctl,
	.listen =	sock_no_listen,
	.shutdown =	sock_no_shutdown,
	.setsockopt =	packet_setsockopt,
	.getsockopt =	packet_getsockopt,
	.sendmsg =	packet_sendmsg,
	.recvmsg =	packet_recvmsg,
	.mmap =		packet_mmap,
	.sendpage =	sock_no_sendpage,
};
 
int sock_no_shutdown(struct socket *sock, int how)
{
	return -EOPNOTSUPP;
}
```
# 修复3
pqarmitage随后提出了新的修改建议，使用sock BPF来禁止sock接收报文。

```
int
if_setsockopt_no_receive(int *sd)
{
	int ret;
	struct sock_filter bpfcode[1] = {
		{0x06, 0, 0, 0},	/* ret #0 - means that all packets will be filtered out */
	};
	struct sock_fprog bpf = {1, bpfcode};
 
	if (*sd < 0)
		return -1;
 
	ret = setsockopt(*sd, SOL_SOCKET, SO_ATTACH_FILTER, &bpf, sizeof(bpf));
	if (ret < 0) {
		log_message(LOG_INFO, "Can't set SO_ATTACH_FILTER option. errno=%d (%m)", errno);
		close(*sd);
		*sd = -1;
	}
 
	return *sd;
}
```

设置后会将相应的filter规则设置到sock上，内核packet_rcv在收到报文后，会通过run_filter(skb, sk, snaplen)来检查相应filter规则，随后丢掉报文。
编译debug版本验证，发现收包队列为0，不会增长。验证有效。

于是达成共识，使用该方案作为最终修复方案。
提交了pull request https://github.com/acassen/keepalived/pull/1982 。
最终合入合入主线。

```
commit 3148dce6666b57adc7854c3bc4227c232d52a1a6
Author: Xing Qingjie <88930741@qq.com>
Date:   Fri Aug 27 08:18:33 2021 -0400
 
    Stop receiving any data on garp socket and ndisc socket
    The rarp broadcast packets would be queued on garp socket, and consume system memory.
    So we used filter stop receiving any data on garp socket and ndisc socket.
    
    [root@cpe ~]# ss -pf link
    Netid Recv-Q Send-Q Local Address:Port Peer Address:Port
    p_raw 204800512 0 rarp:* * users:(("keepalived",pid=2547,fd=12))
    
    Signed-off-by: Xing Qingjie <88930741@qq.com>
 
diff --git a/keepalived/vrrp/vrrp_arp.c b/keepalived/vrrp/vrrp_arp.c
index 3730e8c..2d9b0c9 100644
--- a/keepalived/vrrp/vrrp_arp.c
+++ b/keepalived/vrrp/vrrp_arp.c
@@ -228,6 +228,9 @@ void gratuitous_arp_init(void)
                return;
        }
 
+       /* We don't want to receive any data on this socket */
+       if_setsockopt_no_receive(&garp_fd);
+
        /* Initalize shared buffer */
        garp_buffer = PTR_CAST(char, MALLOC(GARP_BUFFER_SIZE));
 }
diff --git a/keepalived/vrrp/vrrp_ndisc.c b/keepalived/vrrp/vrrp_ndisc.c
index b78b707..6b0e14c 100644
--- a/keepalived/vrrp/vrrp_ndisc.c
+++ b/keepalived/vrrp/vrrp_ndisc.c
@@ -275,6 +275,9 @@ ndisc_init(void)
                log_message(LOG_INFO, "Error %d while registering gratuitous NDISC shared channel", errno);
                return;
        }
+
+       /* We don't want to receive any data on this socket */
+       if_setsockopt_no_receive(&ndisc_fd);
 }
 
 void
```
目前在主线已包含该修复，如果遭遇到该问题困扰的同事可以基于主线出包。

# 后记
之所以这个问题没有早早暴露，是因为socket使用了ETH_P_RARP作为协议，导致只有rarp报文才会触发。大多数情况下的arp报文，并不会触发该问题。
经过这次修复，pqarmitage 修复了这个小问题，使用ETH_P_ARP作为协议。
IPv6的ndic socket也是只用来发送报文，并不接收处理报文，所以也关闭了该socket的接收处理。

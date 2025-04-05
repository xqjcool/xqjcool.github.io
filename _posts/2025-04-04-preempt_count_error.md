---
title: "System malfunction caused by abnormal preempt_count"
date: 2025-04-04
---

# preempt_count 异常导致的系统无法正常工作

最近在做产品的kernel升级，边调试边修复问题，目前基本进入尾声。前几天修复了一个netfilter问题后，编译image加载后系统竟然不能正常工作。
因为改动很小，没想到引发一个大问题，几经波折终于修复了。

## 1. 小小改动引入大问题

内核升级后，需要合并旧内核中自定义的代码。由于本次升级跨度较大，内核源码结构发生了明显变化，导致部分自动合并结果并不正确。
例如，本次 nf_conntrack_standalone.c 文件中的一个错误：
- 旧内核中，ctl_table 数组是按顺序定义的，无需数组下标。
- 新内核采用了 enum nf_ct_sysctl_index 枚举，并使用 [ENUM] = {} 形式明确设置每一项。
自动合并工具虽然保留了原有 nf_sysctl_conn_enable 的条目内容，但遗漏了关键的 [NF_SYSCTL_CT_ENABLE] = 数组下标。
结果是：nf_sysctl_conn_enable 虽然在表中定义了，但没有正确注册进 sysctl，运行时变量没有初始化，无法通过 proc 接口修改，导致 conntrack 功能失效。
修复方法：在 enum nf_ct_sysctl_index 中新增 NF_SYSCTL_CT_ENABLE，并在 nf_ct_sysctl_table[] 中使用 [NF_SYSCTL_CT_ENABLE] = {...} 方式补全定义。

```c
diff --git a/net/netfilter/nf_conntrack_standalone.c b/net/netfilter/nf_conntrack_standalone.c
index 5ca3c86f848..d33d1cbaeda 100644
--- a/net/netfilter/nf_conntrack_standalone.c
+++ b/net/netfilter/nf_conntrack_standalone.c
@@ -620,6 +620,7 @@ enum nf_ct_sysctl_index {
#ifdef CONFIG_LWTUNNEL
        NF_SYSCTL_CT_LWTUNNEL,
#endif
+       NF_SYSCTL_CT_ENABLE,
        __NF_SYSCTL_CT_LAST_SYSCTL,
};
@@ -670,13 +671,6 @@ static struct ctl_table nf_ct_sysctl_table[] = {
                .mode           = 0666,
                .proc_handler   = proc_dointvec,
        },
-       {
-               .procname       = "nf_sysctl_conn_enable",
-               .data           = &nf_sysctl_conn_enable,
-               .maxlen         = sizeof(int),
-               .mode           = 0666,
-               .proc_handler   = proc_dointvec,
-       },
        [NF_SYSCTL_CT_ACCT] = {
                .procname       = "nf_conntrack_acct",
                .data           = &init_net.ct.sysctl_acct,
@@ -969,6 +963,13 @@ static struct ctl_table nf_ct_sysctl_table[] = {
                .proc_handler   = nf_hooks_lwtunnel_sysctl_handler,
        },
#endif
+       [NF_SYSCTL_CT_ENABLE] = {
+               .procname       = "nf_sysctl_conn_enable",
+               .data           = &nf_sysctl_conn_enable,
+               .maxlen         = sizeof(int),
+               .mode           = 0666,
+               .proc_handler   = proc_dointvec,
+       },
        {}
};
```

这部分修改相当简单清晰，开心提交编译版本。没想到运行没多久页面无法访问，console开始卡顿。

## 2. console日志

串口打印了一条故障日志
![image](https://github.com/user-attachments/assets/1fbbfa1d-c42e-4ed6-a5dd-ef97209a73fd)

从日志上看系统处理完软中断后，preempt_count 值异常了。正常情况应该不变的。对应源码如下：

```c
	while ((softirq_bit = ffs(pending))) {
		unsigned int vec_nr;
		int prev_count;

		h += softirq_bit - 1;

		vec_nr = h - softirq_vec;
		prev_count = preempt_count();      //保存执行前的preempt_count

		kstat_incr_softirqs_this_cpu(vec_nr);

		trace_softirq_entry(vec_nr);
		h->action(h);
		trace_softirq_exit(vec_nr);
		if (unlikely(prev_count != preempt_count())) {    //对比现在的preempt_count， 不同则打印异常日志
			pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
			       vec_nr, softirq_to_name[vec_nr], h->action,
			       prev_count, preempt_count());
			preempt_count_set(prev_count);
		}
		h++;
		pending >>= softirq_bit;
	}
```

vec_nr=NET_RX_SOFTIRQ=3, 对应处理函数 `net_rx_action` 。也就是说在收包中断处理中出的问题。
又通过打印发现问题发生在vmxnet虚拟网卡驱动的收包函数 `vmxnet3_poll_rx_only` 处理中。

这样一步一步深入，太慢了，整个协议栈处理太庞大。必须改变策略。

## preempt_count 变化原因

回头查看变化前后的值， 0x7fffff00 = (0x00000100 - 0x00000200) 回绕后再将最高位bit置0得到。因为preempt_count最高位永远置0。

```c
#define PREEMPT_NEED_RESCHED	0x80000000
static __always_inline int preempt_count(void)
{
	return raw_cpu_read_4(__preempt_count) & ~PREEMPT_NEED_RESCHED;  //取整型值，再将最高位bit置0.
}
```

而 - 0x00000200 正好对应 `local_bh_enable` 操作。

```c
#define PREEMPT_BITS	8
#define PREEMPT_SHIFT	0

#define SOFTIRQ_SHIFT	(PREEMPT_SHIFT + PREEMPT_BITS)      //8
#define SOFTIRQ_OFFSET	(1UL << SOFTIRQ_SHIFT)            //0x00000100
#define SOFTIRQ_DISABLE_OFFSET	(2 * SOFTIRQ_OFFSET)      //0x00000200
static inline void local_bh_enable(void)
{
	__local_bh_enable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);  //伪代码： preempt_count -= 0x00000200
}

```

也就是说在软中断收包处理中，某个函数多调用了一次 `local_bh_enable`， 导致 preempt_count值回绕后异常。

于是在`local_bh_enable`中添加了 0x7fffff00的异常判断和打印。
调试后发现在 `tcp_rcv_state_process` 中调用 `local_bh_enable` 后 `preempt_count` 值异常。进一步， 在调用 `acceptable = icsk->icsk_af_ops->conn_request(sk, skb) >= 0` 之前是 0x00000300,之后是0x0000100。

```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
	const struct tcphdr *th = tcp_hdr(skb);
	struct request_sock *req;
	int queued = 0;
	bool acceptable;
	SKB_DR(reason);

	switch (sk->sk_state) {
	case TCP_CLOSE:
		SKB_DR_SET(reason, TCP_CLOSE);
		goto discard;

	case TCP_LISTEN:
		if (th->ack)
			return 1;

		if (th->rst) {
			SKB_DR_SET(reason, TCP_RESET);
			goto discard;
		}
		if (th->syn) {
			if (th->fin) {
				SKB_DR_SET(reason, TCP_FLAGS);
				goto discard;
			}
			/* It is possible that we process SYN packets from backlog,
			 * so we need to make sure to disable BH and RCU right there.
			 */
			rcu_read_lock();
			local_bh_disable();
      //此处 preempt_count=0x300
			acceptable = icsk->icsk_af_ops->conn_request(sk, skb) >= 0;
      //此处 preempt_count=0x100
			local_bh_enable();
      //此处 preempt_count=0x7fffff00  异常
			rcu_read_unlock();

			if (!acceptable)
				return 1;
			consume_skb(skb);
			return 0;
		}
		SKB_DR_SET(reason, TCP_FLAGS);
		goto discard;
//省略无关
	}
	return 0;
}
```

从代码中我们可以看出， `icsk->icsk_af_ops->conn_request(sk, skb)` 处理中多执行了一次 `local_bh_enable` 导致 preempt_count 从0x300降到0x100，继而导致后面的异常。
那么这个错误的 `local_bh_enable` 是在哪个函数中执行的呢？因为这个执行并没有直接触发异常，而如果判断preempt_count == 0x100， 会导致干扰打印太多。
想到添加一个percpu_debug变量，在 `icsk->icsk_af_ops->conn_request(sk, skb)` 执行前设置，后面在 `local_bh_enable` 中判断， percpu_debug打开，并且preempt_count从0x300 --> 0x100则打印异常栈。

## Linux LOCKDEP调试选项

正在添加相关调试信息时，想到有些锁例如read_lock_bh/read_unlock_bh,spin_lock_bh/spin_unlock_bh等也会调用 local_bh_disable/local_bh_enable。不如把对应的调试开关打开，说不定有效果。
于是把相关CONFIG启用。

```bash
CONFIG_LOCKDEP=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_PROVE_LOCKING=y
```

percpu_debug相关添加完毕，CONFIG也启用。image加载后，很快加打印了一个 `WARNING: bad unlock balance detected!`

```bash
[  131.844041] =====================================
[  131.844044] WARNING: bad unlock balance detected!    //告警信息： 检测到加锁解锁不平衡
[  131.844046] 6.1 #14 Tainted: G           O      
[  131.844049] -------------------------------------
[  131.844051] sshd/3725 is trying to release lock (rcu_read_lock_bh) at:      //尝试去释放 rcu_read_lock_bh的锁
[  131.844057] [<ffffffff80bf99ee>] ip_finish_output2+0x11e/0xf00                  //在ip_finish_output2+0x11e进行了 rcu_read_unlock_bh 操作
[  132.004320] CPU: 2 PID: 4714 Comm: lb Kdump: loaded Tainted: G           O       6.1 #14
[  132.060566] but there are no more locks to release!                        //当前没有持有 rcu_read_lock_bh的锁， 无锁可释放
[  132.060569] 
[  132.060569] other info that might help us debug this:
[  132.116832] Hardware name: Default string Default string/Default string, BIOS 5.14 03/30/2021
[  132.172058] 6 locks held by sshd/3725:    //有6个地方进行了rcu_read_lock加锁
[  132.756342]  #0: ffff8881000514c8 (&mm->mmap_lock){++++}-{3:3}, at: vm_mmap_pgoff+0x74/0x140
[  132.857316]  #1: ffffffff81b52fc0 (rcu_read_lock){....}-{1:2}, at: netif_receive_skb_list_internal+0xdd/0x410
[  132.975981]  #2: ffffffff81b52fc0 (rcu_read_lock){....}-{1:2}, at: ip_local_deliver_finish+0x94/0x1f0
[  133.086324]  #3: ffffffff81b52fc0 (rcu_read_lock){....}-{1:2}, at: tcp_rcv_state_process+0x3de/0xfd0
[  133.195630]  #4: ffffffff81b52fc0 (rcu_read_lock){....}-{1:2}, at: tcp_v4_send_synack+0x1a4/0x450
[  133.230860] Call Trace:
[  133.301812]  #5: ffffffff81b52fc0
[  133.331037]  <IRQ>
[  133.331041]  dump_stack_lvl+0x68/0x83
[  133.370661]  (rcu_read_lock){....}-{1:2}, at: ip_finish_output2+0xc0/0xf00    //在ip_finish_output2+0xc0进行了 rcu_read_lock 加锁
```

从告警日志打印，清晰的获知到 `ip_finish_output2` 函数执行 `rcu_read_unlock_bh`取释放锁，但是没有持有(即没有调用过rcu_read_lock_bh的锁)。

查看源码，发现自动merge的代码中使用了 `rcu_read_unlock_bh` ，这是因为旧版kernel代码使用的是 `rcu_read_lock_bh/rcu_read_unlock_bh`,但新kernel改为了 `rcu_read_lock/rcu_read_unlock`。
但是自动merge不能识别这些变化，导致这里出现了问题。

```c
int ip_finish_output2(struct net *net, struct sock *sk, struct sk_buff *skb)
{
//省略无关

	rcu_read_lock();
+#ifdef CONFIG_NF_CT_REVERSE_ROUTING
+	if (nf_ct_reverse_skb_ready(skb)) {
+		res = nf_ct_reverse_skb_xmit(skb);
+		rcu_read_unlock_bh();        //这里还用的是旧版的rcu_read_unlock_bh去释放
+		return res;
+	}
+#endif
rcu_read_unlock();
//省略无关
}

```

## 后记

Linux中有很多debug CONFIG选项，在调试中可以提供很大帮助。所以还是要熟悉相关DEBUG CONFIG选项。
linxu自动merge代码还是不够智能，埋了很多的坑 :-(


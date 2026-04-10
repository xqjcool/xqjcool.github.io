---
title: "[Performance] 从“自旋锁”到“无锁原子化”：一次性能优化的解剖"
date: 2026-04-10
---

# 从“自旋锁”到“无锁原子化”：一次性能优化的解剖

## 1. QA上报开启虚拟域后性能巨幅下降

QA上报说，不开启虚拟域时性能正常，开启后，性能急剧下降，不到关闭时性能的20% 。 如此大的性能下降必然是有明显的瓶颈限制了性能。于是拿到环境，进行了性能分析。

### 1.1 top显示，性能都消耗在软中断处理中了。

```
Mem: 7946832K used, 123526396K free, 4303094K shrd, 184720K buff, 4492132K cach
CPU0:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU1:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU2:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU3:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU4:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU5:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU6:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU7:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU8:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU9:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU10:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU11:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU12:  0.0% usr 16.6% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq 83.3% sirq
CPU13:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU14:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU15:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU16:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU17:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU18:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU19:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU20:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU21:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU22:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU23:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU24:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU25:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU26:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU27:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU28:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU29:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU30:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
CPU31:  0.0% usr  0.0% sys  0.0% nic  0.0% idle  0.0% io  0.0% irq  100% sirq
```

### 1.2 perf top 显示性能主要消耗在queued_spin_lock_slowpath，说明有严重的锁竞争

```
   PerfTop:  130619 irqs/sec  kernel:99.6%  exact: 100.0% [4000Hz cycles:ppp],  (all, 32 CPUs)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

    77.48%  [kernel]       [k] queued_spin_lock_slowpath
     2.35%  [kernel]       [k] do_raw_spin_lock
     1.50%  [kernel]       [k] ipt_do_table
     1.45%  [kernel]       [k] virtual_space_l4_traffic_rlimit
     0.86%  [kernel]       [k] i40e_napi_poll
     0.83%  [kernel]       [k] fib_table_lookup
     0.61%  [kernel]       [k] ip_vs_conn_get
     0.55%  [kernel]       [k] ip_rcv
     0.47%  [kernel]       [k] __dev_queue_xmit
```


## 2 深入分析

为了获取更详细的信息，我使用 `perf record` 对系统运行时的 CPU 进行采样记录，并通过 `perf report` 对采样数据进行分析，查看热点函数和调用栈分布。
发现问题集中在 `virtual_space_l4_traffic_rlimit` 的虚拟域限速处理上。但是设备上并没有限速配置，也就是说 `virtual_space_l4_traffic_rlimit` 限速逻辑空跑就已经导致了性能下降。



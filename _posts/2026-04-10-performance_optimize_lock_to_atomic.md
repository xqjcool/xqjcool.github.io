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

### 2.1 perf record采样分析

为了获取更详细的信息，我使用 `perf record` 对系统运行时的 CPU 进行采样记录，并通过 `perf report` 对采样数据进行分析，查看热点函数和调用栈分布。
发现问题集中在 `virtual_space_l4_traffic_rlimit` 的虚拟域限速处理上。但是设备上并没有限速配置，也就是说 `virtual_space_l4_traffic_rlimit` 限速逻辑空跑就已经导致了性能下降。

perf report文件中 对 queued_spin_lock_slowpath 的调用主要由 virtual_space_l4_traffic_rlimit 发起，超多并发导致了锁竞争 。

部分report如下

```
  --3.01%--__netif_receive_skb
            |          
             --3.01%--__netif_receive_skb_core
                       |          
                        --2.97%--ip_rcv
                                  |          
                                   --2.80%--ip_rcv_finish
                                             |          
                                             |--1.83%--ip_forward
                                             |          |          
                                             |           --1.82%--nf_hook_slow
                                             |                     |          
                                             |                      --1.82%--ip_vs_reply4
                                             |                                |          
                                             |                                 --1.82%--ip_vs_out.part.0
                                             |                                           |          
                                             |                                            --1.79%--handle_response.constprop.0
                                             |                                                      |          
                                             |                                                       --1.67%--virtual_space_l4_traffic_rlimit
                                             |                                                                 |          
                                             |                                                                  --1.64%--__lock_text_start
                                             |                                                                            |          
                                             |                                                                             --1.63%--do_raw_spin_lock
                                             |                                                                                       |          
                                             |                                                                                        --1.61%--queued_spin_lock_slowpath
                                             |          
                                              --0.86%--ip_local_deliver
                                                        |          
                                                         --0.86%--nf_hook_slow
                                                                   |          
                                                                    --0.83%--ip_vs_remote_request4
                                                                              |          
                                                                               --0.83%--ip_vs_in.part.0
                                                                                         |          
                                                                                          --0.82%--ip_vs_in_cont
                                                                                                    |          
                                                                                                     --0.76%--virtual_space_l4_traffic_rlimit
                                                                                                               |          
                                                                                                                --0.75%--__lock_text_start
                                                                                                                          |          
                                                                                                                           --0.75%--do_raw_spin_lock
                                                                                                                                     |          
                                                                                                                                      --0.73%--queued_spin_lock_slowpath
```


### 2.2 分析 virtual_space_l4_traffic_rlimit

#### 2.2.1 第一个spin lock net->virtual_space_rlimit_hash_lock
查看 virtual_space_l4_traffic_rlimit 函数，我们发现，它上来就直接调用 spin_lock_irqsave(&net->virtual_space_rlimit_hash_lock, flags), 整个函数处理都在自旋锁的保护下。
而 virtual_space_l4_traffic_rlimit 又是在datapath，导致每个packet都要竞争spin lock。多核并发处理下，锁竞争严重影响到了包处理性能。

经分析这个lock 只是为了保证 discard计数的并发更新。实际上我们可以考虑对计数使用atomic原子操作，这样此处就不需要spin lock了。

#### 2.2.2 第二个 spin lock rl->lock

在进一步处理中，函数尝试获取 raw_spin_trylock_irqsave(&rl->lock, flags) 来进行后续操作。同样也严重影响到性能。
经分析这个lock 只是为了保证 printed/missed 的更新。我们同样可以考虑对计数使用atomic原子操作，这样此处就不需要spin lock了。


## 3. 优化难点

虽然一些统计计数，我们可以用原子atomic操作替代，避免使用spin lock。 
但是限速窗口刷新必须保证只能由一个CPU执行，这是一个难点。为了避免窗口刷新并发，我们添加了一个refreshing 标志。
通过原子 atomic_cmpxchg 保证只有一个CPU 进入到刷新处理。其他CPU会短暂循环等待，因为刷新默认是1秒，不像per packet处理那么频繁，并发概率较低，所以几乎可以忽略不记。

```
    for (;;) {
			/* 其他CPU在这里获取到新的begin */
            start = READ_ONCE(rl->begin);
            if (start && time_before(jiffies, start + interval))
                    return;

			/* 第一个刷新的 set refreshing, 后面的等待刷新完成 */
            if (atomic_cmpxchg(&rl->refreshing, 0, 1)) {
                    cpu_relax();
                    continue;
            }

            /* 只有一个CPU可以走到这一步 */ 
            atomic64_set(&rl->printed, 0);
            atomic_set(&rl->missed, 0);
			/* 更新 begin */
            WRITE_ONCE(rl->begin, jiffies);

            /* 刷新完成 unset refressing. */
            smp_store_release(&rl->refreshing.counter, 0);
    }
```

有了这个处理就能保证 窗口刷新没有冲突。配合atomic 原子操作更新计数。我们就可以将原因spin lock删除，实现无锁处理。

## 4. 性能验证

debug image交由QA验证，性能大幅提升，接近 不开启 virtual space时的性能。
优化前 开启virtual space 是不开启 virtual space的 20% ，约为1/5 。 
优化后 开启virtual space 是不开启 virtual space的 95% 。
优化后 开启virtual space的性能是 优化前 开启virtual space 性能的 5倍。

### 4.1 查看 perf top

再次对比 perf top，已经没有 queued_spin_lock_slowpath 了

```
   PerfTop:  128592 irqs/sec  kernel:99.0%  exact: 100.0% lost: 0/0 drop: 0/0 [4000Hz cycles],  (all, 32 CPUs)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

     9.13%  [kernel]       [k] ipt_do_table
     7.00%  [kernel]       [k] __dev_queue_xmit
     4.21%  [kernel]       [k] fib_table_lookup
     3.20%  [kernel]       [k] i40e_napi_poll
     2.39%  [kernel]       [k] i40e_lan_xmit_frame
     2.11%  [kernel]       [k] handle_response
```

## 5. 后记

spin lock能够解决并发问题，但在某些场合因为并发冲突，导致性能下降。
原子操作、无锁处理才是更优的解决方案。

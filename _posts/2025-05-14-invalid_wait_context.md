---
title: "BUG: Invalid wait context"
date: 2025-05-14
---

# BUG：非法的等待上下文

## 1. 问题起因

最近在升级产品内核。为了让问题尽早暴露，并方便定位，开启了很多DEBUG选项。

```bash
CONFIG_DEBUG_LOCK_ALLOC=y    //其中一个配置项
```

在测试中遇到了异常日志打印：

```bash
May 13 15:16:32 kern.warn kernel: [ 1880.242388] =============================
May 13 15:16:32 kern.warn kernel: [ 1880.242429] [ BUG: Invalid wait context ]
May 13 15:16:32 kern.warn kernel: [ 1880.242469] 6.1 #3 Tainted: G           O      
May 13 15:16:32 kern.warn kernel: [ 1880.242517] -----------------------------
May 13 15:16:32 kern.warn kernel: [ 1880.242557] tc/6787 is trying to lock:
May 13 15:16:32 kern.warn kernel: [ 1880.242595] ffff888100e66298 (&dev->mutex){....}-{3:3}, at: netif_freeze_queues+0x4b/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.242677] other info that might help us debug this:
May 13 15:16:32 kern.warn kernel: [ 1880.242724] context-{4:4}
May 13 15:16:32 kern.warn kernel: [ 1880.242754] 2 locks held by tc/6787:
May 13 15:16:32 kern.warn kernel: [ 1880.242792]  #0: ffffffff81a42e08 (rtnl_mutex){+.+.}-{3:3}, at: rtnetlink_rcv_msg+0x153/0x5b0
May 13 15:16:32 kern.warn kernel: [ 1880.242874]  #1: ffff888100ef1438 (&dev->tx_global_lock){+.-.}-{2:2}, at: dev_deactivate_many+0xbb/0x410
May 13 15:16:32 kern.warn kernel: [ 1880.242962] stack backtrace:
May 13 15:16:32 kern.warn kernel: [ 1880.242996] CPU: 0 PID: 6787 Comm: tc Kdump: loaded Tainted: G           O       6.1 #3
May 13 15:16:32 kern.warn kernel: [ 1880.243069] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
May 13 15:16:32 kern.warn kernel: [ 1880.243159] Call Trace:
May 13 15:16:32 kern.warn kernel: [ 1880.243188]  <TASK>
May 13 15:16:32 kern.warn kernel: [ 1880.243216]  dump_stack_lvl+0x76/0xaa
May 13 15:16:32 kern.warn kernel: [ 1880.243257]  dump_stack+0x10/0x16
May 13 15:16:32 kern.warn kernel: [ 1880.243293]  __lock_acquire.cold+0x420/0x474
May 13 15:16:32 kern.warn kernel: [ 1880.243339]  lock_acquire+0xc2/0x2b0
May 13 15:16:32 kern.warn kernel: [ 1880.243379]  ? netif_freeze_queues+0x4b/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.243422]  ? lock_acquire+0xc2/0x2b0
May 13 15:16:32 kern.warn kernel: [ 1880.243462]  ? dev_deactivate_many+0xbb/0x410
May 13 15:16:32 kern.warn kernel: [ 1880.243507]  _raw_spin_lock+0x2f/0x50
May 13 15:16:32 kern.warn kernel: [ 1880.243547]  ? netif_freeze_queues+0x4b/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.243591]  netif_freeze_queues+0x4b/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.243633]  ? dev_deactivate_many+0x1f/0x410
May 13 15:16:32 kern.warn kernel: [ 1880.243677]  dev_deactivate_many+0xc3/0x410
May 13 15:16:32 kern.warn kernel: [ 1880.243721]  dev_deactivate+0x3c/0x70
May 13 15:16:32 kern.warn kernel: [ 1880.243760]  qdisc_graft+0x5d9/0x800
May 13 15:16:32 kern.warn kernel: [ 1880.243800]  tc_modify_qdisc+0x68f/0xa70
May 13 15:16:32 kern.warn kernel: [ 1880.243843]  rtnetlink_rcv_msg+0x17d/0x5b0
May 13 15:16:32 kern.warn kernel: [ 1880.243885]  ? lock_acquire+0xc2/0x2b0
May 13 15:16:32 kern.warn kernel: [ 1880.243924]  ? find_held_lock+0x31/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.243964]  ? netlink_deliver_tap+0xfb/0x520
May 13 15:16:32 kern.warn kernel: [ 1880.244010]  ? rtnl_fdb_dump+0x2d0/0x2d0
May 13 15:16:32 kern.warn kernel: [ 1880.244050]  netlink_rcv_skb+0x57/0x110
May 13 15:16:32 kern.warn kernel: [ 1880.244092]  rtnetlink_rcv+0x15/0x20
May 13 15:16:32 kern.warn kernel: [ 1880.244129]  netlink_unicast+0x1ab/0x290
May 13 15:16:32 kern.warn kernel: [ 1880.244171]  netlink_sendmsg+0x22d/0x480
May 13 15:16:32 kern.warn kernel: [ 1880.244212]  ? netlink_unicast+0x290/0x290
May 13 15:16:32 kern.warn kernel: [ 1880.244265]  ____sys_sendmsg+0x230/0x260
May 13 15:16:32 kern.warn kernel: [ 1880.244307]  ___sys_sendmsg+0x96/0xd0
May 13 15:16:32 kern.warn kernel: [ 1880.244347]  __sys_sendmsg+0x7d/0xd0
May 13 15:16:32 kern.warn kernel: [ 1880.244385]  __x64_sys_sendmsg+0x1d/0x30
May 13 15:16:32 kern.warn kernel: [ 1880.244426]  do_syscall_64+0x37/0x90
May 13 15:16:32 kern.warn kernel: [ 1880.244465]  entry_SYSCALL_64_after_hwframe+0x64/0xce
May 13 15:16:32 kern.warn kernel: [ 1880.244515] RIP: 0033:0x7f0ba008dbb0
May 13 15:16:32 kern.warn kernel: [ 1880.244558] Code: 00 f7 d8 64 89 02 b8 ff ff ff ff eb bb 66 2e 0f 1f 84 00 00 00 00 00 0f 1f 00 80 3d 79 46 0d 00 00 74 17 b8 2e 00 00 00 0f 05 <48> 3d 00 f0 ff ff 77 58 c3 0f 1f 80 00 00 00 00 48 83 ec 28 89 54
May 13 15:16:32 kern.warn kernel: [ 1880.244708] RSP: 002b:00007ffea3bc3548 EFLAGS: 00000202 ORIG_RAX: 000000000000002e
May 13 15:16:32 kern.warn kernel: [ 1880.244778] RAX: ffffffffffffffda RBX: 000000006823c4c2 RCX: 00007f0ba008dbb0
May 13 15:16:32 kern.warn kernel: [ 1880.244842] RDX: 0000000000000000 RSI: 00007ffea3bc35a0 RDI: 0000000000000007
May 13 15:16:32 kern.warn kernel: [ 1880.244918] RBP: 000055cd2dc273a0 R08: 0000000000000010 R09: 0000000000000000
May 13 15:16:32 kern.warn kernel: [ 1880.244986] R10: 00007f0b9ffa1150 R11: 0000000000000202 R12: 00007ffea3bc35a0
May 13 15:16:32 kern.warn kernel: [ 1880.245048] R13: 000055cd2dc08b6e R14: 0000000000000000 R15: 0000000000000000
May 13 15:16:32 kern.warn kernel: [ 1880.245111]  </TASK>

```

## 2. 初步分析

单从日志上看， 象是 `netif_freeze_queues+0x4b/0x90` 对 `&dev->mutex` 进行上锁。 但是这是在原子上下文，不能使用可休眠的锁。于是触发了异常日志。

但是 `netif_freeze_queues+0x4b` 实际上并没有 对 `&dev->mutex` 上mutex锁，而是对 `&txq->_xmit_lock` 上 spin_lock锁。

```c
static void netif_freeze_queues(struct net_device *dev)
{
        unsigned int i;
        int cpu;

        cpu = smp_processor_id();
        for (i = 0; i < dev->num_tx_queues; i++) {
                struct netdev_queue *txq = netdev_get_tx_queue(dev, i);

                /* We are the only thread of execution doing a
                 * freeze, but we have to grab the _xmit_lock in
                 * order to synchronize with threads which are in
                 * the ->hard_start_xmit() handler and already
                 * checked the frozen bit.
                 */
                __netif_tx_lock(txq, cpu);    //here
                set_bit(__QUEUE_STATE_FROZEN, &txq->state);
                __netif_tx_unlock(txq);
        }
}

static inline void __netif_tx_lock(struct netdev_queue *txq, int cpu)
{
        spin_lock(&txq->_xmit_lock);          //here
        /* Pairs with READ_ONCE() in __dev_queue_xmit() */
        WRITE_ONCE(txq->xmit_lock_owner, cpu);
}
```

什么情况，内核怎么会将 spin_lock(&txq->_xmit_lock) 与 mutex_lock(&dev->mutex) 混淆了？？？

## 3. 抽丝剥茧

那我们就看看日志中的lockdep_map的详细信息吧。
用crash工具实时调试内核。

```bash
crash> lockdep_map -x ffff888100e66298        //日志中提供的地址
struct lockdep_map {
  key = 0xffffffff81cfcc20 <__lockdep_no_validate__>,    //这个key不大对劲
  class_cache = {0xffffffff81f08900 <lock_classes+47232>, 0x0},
  name = 0xffffffff814f2f17 "&txq->_xmit_lock",    //确实是 对 &txq->_xmit_lock 进行上锁
  wait_type_outer = 0x0,
  wait_type_inner = 0x2,
  lock_type = 0x0
}

crash> raw_spinlock_t -x ffff888100e66280  //对应的spin_lock
struct raw_spinlock_t {
  raw_lock = {
    {
      val = {
        counter = 0x0
      },
      {
        locked = 0x0,
        pending = 0x0
      },
      {
        locked_pending = 0x0,
        tail = 0x0
      }
    }
  },
  magic = 0xdead4ead,
  owner_cpu = 0xffffffff,
  owner = 0xffffffffffffffff,
  dep_map = {
    key = 0xffffffff81cfcc20 <__lockdep_no_validate__>,
    class_cache = {0xffffffff81f08900 <lock_classes+47232>, 0x0},
    name = 0xffffffff814f2f17 "&txq->_xmit_lock",
    wait_type_outer = 0x0,
    wait_type_inner = 0x2,
    lock_type = 0x0
  }
}

```

从调试信息可以看出， 确实是对 `&txq->_xmit_lock` 上锁， 那么为什么会出现 `&dev->mutex` 呢？

从代码上看，在 `__lock_acquire` 中会从 class_cache中取得 class来做一些检查。

调用路径：
spin_lock-->raw_spin_lock-->_raw_spin_lock-->__raw_spin_lock-->spin_acquire-->**lock_acquire_exclusive**-->lock_acquire-->__lock_acquire


```c
static int __lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                          int trylock, int read, int check, int hardirqs_off,
                          struct lockdep_map *nest_lock, unsigned long ip,
                          int references, int pin_count)
{
        struct task_struct *curr = current;
        struct lock_class *class = NULL;
        struct held_lock *hlock;
        unsigned int depth;
        int chain_head = 0;
        int class_idx;
        u64 chain_key;

        if (unlikely(!debug_locks))
                return 0;

        if (!prove_locking || lock->key == &__lockdep_no_validate__)
                check = 0;

        if (subclass < NR_LOCKDEP_CACHING_CLASSES)
                class = lock->class_cache[subclass];      //here
        /*
         * Not cached?
         */
        if (unlikely(!class)) {
                class = register_lock_class(lock, subclass, 0);  //or here
                if (!class)
                        return 0;
        }
//省略无关
        hlock = curr->held_locks + depth;

        class_idx = class - lock_classes;

        hlock->class_idx = class_idx;
}

```

我们检查下 lockdep_map的 class

```bash
crash> lock_class -x 0xffffffff81f08900
struct lock_class {
  hash_entry = {
    next = 0x0,
    pprev = 0xffffffff81f290c0 <lock_classes+180288>
  },
  lock_entry = {
    next = 0xffffffff81f089d0 <lock_classes+47440>,
    prev = 0xffffffff81f08850 <lock_classes+47056>
  },
  locks_after = {
    next = 0xffffffff81f08920 <lock_classes+47264>,
    prev = 0xffffffff81f08920 <lock_classes+47264>
  },
  locks_before = {
    next = 0xffffffff81f08930 <lock_classes+47280>,
    prev = 0xffffffff81f08930 <lock_classes+47280>
  },
  key = 0xffffffff81cfcc20 <__lockdep_no_validate__>,    //key也是有点奇怪
  subclass = 0x0,
  dep_gen_id = 0x0,
  usage_mask = 0x100,
  usage_traces = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0xffffffff82158a98 <stack_trace+72056>, 0x0},
  name_version = 0x1,
  name = 0xffffffff814e882a "&dev->mutex",    //真凶出现
  wait_type_inner = 0x3,
  wait_type_outer = 0x0,
  lock_type = 0x0
}
```

解析到这里，也就明白为什么出现 `&dev->mutex` 相关的错误日志的直接原因了。但是为什么 `&txq->_xmit_lock` 的class 是 `&dev->mutex` 呢？

## 4. 继续深入

在前面的分析中我们注意到2个疑点， 一个是 `&dev->mutex`, 另一个是 `__lockdep_no_validate__` 。

### 4.1  `&dev->mutex` 的初始化：

```c
void device_initialize(struct device *dev)
{
//省略无关
        mutex_init(&dev->mutex);                  //初始化mutex
        lockdep_set_novalidate_class(&dev->mutex);  //设置novalidate
//省略无关
}

#define mutex_init(mutex)                                               \
do {                                                                    \
        static struct lock_class_key __key;                             \
                                                                        \
        __mutex_init((mutex), #mutex, &__key);                          \
} while (0)


#define lockdep_set_novalidate_class(lock) \
        lockdep_set_class_and_name(lock, &__lockdep_no_validate__, #lock)

```

第一个是初始化 mutex `&dev->mutex` , key是一个静态全局变量。
调用路径：
mutex_init-->__mutex_init-->debug_mutex_init->**lockdep_init_map_wait**-->lockdep_init_map_waits-->lockdep_init_map_type-->

第二个时设置novalidate，其实就是将mutex的 key替换成 __lockdep_no_validate__ 全局变量，最终调用 lockdep_init_map_type，其他参数值不变。



### 4.2 `&dev->mutex` 的上锁

mutex_lock执行路径：
mutex_lock-->mutex_lock_nested-->__mutex_lock-->__mutex_lock_common-->mutex_trylock-->mutex_acquire-->**lock_acquire_exclusive**-->lock_acquire-->__lock_acquire-->[first time]register_lock_class

在 **lock_acquire_exclusive** 之后的路径和 spin_lock相同。初次使用时会对class进行注册。

```c
//伪代码
static struct lock_class *
register_lock_class(struct lockdep_map *lock, unsigned int subclass, int force)
{
  //1. 在classhash_table中查找 key对应的calss， 条件是 class->key == key
  //2. 如果找到，直接返回对应的class
  //3. 如果不存在，则新建一个class，并初始化
  //4. 添加到classhash_table中
  //5. lock->class_cache[0] = class; 返回class
}
```

所以在mutex_lock(&dev->mutex) 初次执行时，触发了 register_lock_class，创建了一个 key==__lockdep_no_validate__的class，并添加到哈希表中。
后续直接就从 lock->class_cache[0] 中取class。

## 5. 黎明曙光

熟悉了 `&dev->mutex` 的相关操作后， 回过头来看 `&txq->_xmit_lock`。

### 5.1 `&txq->_xmit_lock`的初始化

```c
static int netif_alloc_netdev_queues(struct net_device *dev)
{
//省略无关  
        netdev_for_each_tx_queue(dev, netdev_init_one_queue, NULL);
//省略无关
        return 0;
}

static void netdev_init_one_queue(struct net_device *dev,
                                  struct netdev_queue *queue, void *_unused)
{
        /* Initialize queue lock */
        spin_lock_init(&queue->_xmit_lock);
//省略无关
}

# define spin_lock_init(lock)                                   \
do {                                                            \
        static struct lock_class_key __key;                     \
                                                                \
        __raw_spin_lock_init(spinlock_check(lock),              \
                             #lock, &__key, LD_WAIT_CONFIG);    \
} while (0)


```

调用路径：
spin_lock_init-->__raw_spin_lock_init-->**lockdep_init_map_wait**-->lockdep_init_map_waits-->lockdep_init_map_type

可以看到初始化的 key是个静态全局变量， 但我们实际观察到的是 __lockdep_no_validate__ 。

继续搜索，发现在代码中对特定设备的 `&txq->_xmit_lock` 进行了 novalidate处理。

```c

static void __init imq_dev_set_lockdep_one(struct net_device *dev,
                            struct netdev_queue *txq, void *arg)
{
        lockdep_set_novalidate_class(&txq->_xmit_lock);
}
```

这个会将lockdep_map的key 替换成 __lockdep_no_validate__ 。

看到这里，想必你已经基本明白问题的最终原因了。不过不急，我们把最后一步做完。

### 5.2 `&txq->_xmit_lock`的上锁

在第3节我们分析了部分 spin_lock调用路径的 `__lock_acquire` 代码。当时只是为了查看 class。而这次我们看看class 从哪里来的。

调用路径：
spin_lock-->raw_spin_lock-->_raw_spin_lock-->__raw_spin_lock-->spin_acquire-->lock_acquire_exclusive-->lock_acquire-->__lock_acquire-->register_lock_class

当第一次调用spin_lock时， 由于 lockdep_map.class_cache[subclass] == NULL, 进入 register_lock_class。 使用 key 为 __lockdep_no_validate__ 在哈希表中进行查找。
**注意** 因为mutex_lock(&dev->mutex) 已经执行过， 所以 哈希表中已经有 key 为 __lockdep_no_validate__ 的表项。
所以在伪代码的第2步，就获取到class了，且这个class的name是 `&dev->mutex`。随后 lock->class_cache[0] = class 。
也就是我们通过crash工具查看到的结果。

```c
//伪代码
static struct lock_class *
register_lock_class(struct lockdep_map *lock, unsigned int subclass, int force)
{
  //1. 在classhash_table中查找 key对应的calss， 条件是 class->key == key
  //2. 如果找到，跳转第5步
  //3. 如果不存在，则新建一个class，并初始化
  //4. 添加到classhash_table中
  //5. lock->class_cache[0] = class; 返回class
}
```

## 6. 水落石出

分析到这里，结果已然跃于纸上。
问题在于 `&dev->mutex` 和 `&txq->_xmit_lock` 都使用了 `lockdep_set_novalidate_class` 将其key值设为相同的 __lockdep_no_validate__ 。而 `&dev->mutex` 的上锁操作较早，将 `&dev->mutex` 的class添加到哈希表中。
后续 `&txq->_xmit_lock` 的上锁操作，就会根据 key为 __lockdep_no_validate__ 查找到 `&dev->mutex` 的class, 导致随后的坚持出现问题， 打印异常日志。


## 7. 后记

这个问题是锁的lockdep_map相关debug处理时出现的。 如果不开启对应DEBUG选项， 则不会有这个错误打印。

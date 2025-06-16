---
title: "The Principles and Implementation of the Linux CFS Scheduler"
date: 2025-06-09
---

# linux CFS 调度原理与实现

CFS(Completely Fair Scheduler) 在linux 2.6.23版本进入主线。

## 1. 基础架构图

![Untitled Diagram drawio](https://github.com/user-attachments/assets/d9d96fda-5252-47cb-9f07-dd91f0661131)

## 2. 调度器概述

### 2.1 调度器种类

目前linux中有3类主要的调度器：

- 截止时间 调度器
- 实时 调度器
- 完全公平 调度器

内核中per-CPU runqueue运行队列数据结构中定义了相应的运行队列。
也定义了对应的调度类。

```c
struct rq {
//部分省略
	struct cfs_rq		cfs;
	struct rt_rq		rt;
	struct dl_rq		dl;
//部分省略
}

const struct sched_class fair_sched_class;
const struct sched_class rt_sched_class;
const struct sched_class dl_sched_class;

//两个特殊的调度类
const struct sched_class idle_sched_class;
const struct sched_class stop_sched_class;
```

- idle_sched_class 用于在没有任务时，运行idle线程
- stop_sched_class 用于执行"migration/x"线程。

### 2.2 调度策略

目前linux中有 6 种调度策略，它们对应不同的调度器，后面会有个表格总结。

- 普通调度
- 先进先出调度
- 轮转调度
- 批量调度
- 空闲调度
- 截止时间调度

从内核的宏定义可以看到

```c
#define SCHED_NORMAL		0
#define SCHED_FIFO		1
#define SCHED_RR		2
#define SCHED_BATCH		3
/* SCHED_ISO: reserved but not implemented yet */
#define SCHED_IDLE		5
#define SCHED_DEADLINE		6
```

### 2.3 优先级和nice

- 内核中的优先级

deadline的优先级就是 -1
实时任务的优先级范围是 [0,99],
普通认为的优先级范围是 [100,139]，

- 用户态用nice来设置普通进程的优先级

nice和prio的转化逻辑如下

```c
#define MAX_NICE	19
#define MIN_NICE	-20
#define NICE_WIDTH	(MAX_NICE - MIN_NICE + 1)	//40
#define MAX_RT_PRIO		100

#define MAX_PRIO		(MAX_RT_PRIO + NICE_WIDTH)	//140
#define DEFAULT_PRIO		(MAX_RT_PRIO + NICE_WIDTH / 2)	//120

/*
 * Convert user-nice values [ -20 ... 0 ... 19 ]
 * to static priority [ MAX_RT_PRIO..MAX_PRIO-1 ],
 * and back.
 */
#define NICE_TO_PRIO(nice)	((nice) + DEFAULT_PRIO)		//nice + 120
#define PRIO_TO_NICE(prio)	((prio) - DEFAULT_PRIO)		//prio - 120
```

### 2.4 总结表格

| 调度器 | 调度队列 | 调度类 | 调度策略 | 优先级 | 
| --- | --- | --- | --- | --- |
| 普通调度 | cfs_rq | fair_sched_class | SCHED_NORMAL,SCHED_BATCH | 100~139 |
| 实时调度 | rt_rq | rt_sched_class | SCHED_FIFO,SCHED_RR | 0~99 |
| deadline调度 | dl_rq | dl_sched_class | SCHED_DEADLINE | -1 |

## 3. CFS 完全公平调度器

CFS调度发展到现在，其结构和实现比最初负责了很多，但是核心逻辑基本不变。我们聚焦在核心逻辑，忽略细枝末节，方便尽快理解和掌握CFS调度。

### 3.1 核心结构和相关变量

#### 3.1.1 cfs_rq

```c
/* CFS-related fields in a runqueue */
struct cfs_rq {
//省略无关
	struct rb_root_cached	tasks_timeline;
	struct sched_entity	*curr;
//省略无关
}
```

#### 3.1.2 rb_root_cached

```c
struct rb_root_cached {
	struct rb_root rb_root;
	struct rb_node *rb_leftmost;
};
```

#### 3.1.3 sched_entity

```c
struct sched_entity {
//省略无关
	struct rb_node			run_node;
	u64				vruntime;
//省略无关
}
```

#### 3.1.4 task_struct

```c
struct task_struct {
//省略无关
	struct sched_entity		se;
//省略无关
}
```

### 3.2 核心原理机制

#### 3.2.1 vruntime(虚拟运行时间)

进程运行的每一单位时间都会增加其 vruntime，计算公式考虑调度权重（即 nice 值），nice 值越低（优先级高）vruntime 增长越慢。

#### 3.2.2 红黑树调度策略

所有就绪的调度实体按 vruntime 组织在红黑树中，调度器每次选择 vruntime 最小（最左）的实体执行。
为了实现O(1)操作，增加了一个 leftmode指针，指向红黑树最左的实体。当调度器选择vruntime最小(最左)时，可以直接得到，而不用从root去遍历。

#### 3.2.3 调度逻辑

- 主动：任务调用cond_resched或schedule主动让出时间片。
- 被动：当任务运行的时间超过了其 slice（时间配额），调度器会设置 TIF_NEED_RESCHED 标志。后续在调度点（如返回用户态）检查此标志并触发调度。

**调度点**
以非抢占式(CONFIG_PREEMPTION未设置)为例。
a. 系统调用或异常返回用户态时
b. 中断处理返回用户态时

### 3.3 CFS调度逻辑

![Untitled Diagram drawio](https://github.com/user-attachments/assets/d9d96fda-5252-47cb-9f07-dd91f0661131)

这是个简化后的CFS调度框架。我们基于它来理清CFS调度的基础逻辑。

#### 3.3.1 结构梳理

在多核系统中，每个cpu都有一个调度队列结构 rq，它包含了 CFS调度队列cfs。cfs是CFS调度队列结构，我们目前只关注优化后的红黑树结构tasks_timeline和当前运行实体*curr 。

tasks_timeline 结构包含了一个 rb_root红黑树，还有一个rb_leftmode指针，它指向红黑树中最左侧的节点，也就是vruntime最小的节点。这样做的优点就是实现了O(1)操作，不用去红黑树中遍历查找。

#### 3.3.2 调度方式

- schedule 主动调度(通常设置TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE)

进程等待资源(I/O,锁等)或condition不满足时，调用schedule()让出CPU。

- cond_resched 主动检查调度(不改变任务状态)

条件：抢占计数器是否为0，是否设置了重新调度标志 TIF_NEED_RESCHED 
在不可抢占系统中，为了避免任务执行时间过长，导致其他任务starve。显式请求调度，如果需要，则安全地让出CPU；如果不需要，则什么都不做，进程继续运行。

- 系统调用/异常返回/中断返回(隐式调用schedule)

都是在返回用户态前，对重新调度标志 TIF_NEED_RESCHED 进行检查，如果设置了，则会进入schedule(),

#### 3.3.3 时间片耗尽

系统每个tick运行一次tick_sched_timer，它会调用






---
title: "The Principles and Implementation of the Linux CFS Scheduler"
date: 2025-06-09
---

# linux CFS 调度原理与实现



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

### 2.3 

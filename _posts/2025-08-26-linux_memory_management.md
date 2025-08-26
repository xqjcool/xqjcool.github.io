---
title: ""
date: 2025-08-26
---

# Linux 内存管理框架

## 1. 基础框架图

<img width="419" height="187" alt="image" src="https://github.com/user-attachments/assets/0181b1a9-4834-4231-8d41-87cb4c374141" />

## 2. 核心结构及其关系

### 2.1 pglist_data

struct pglist_data 是 Linux 内核中用于管理 每个 NUMA 节点物理内存 的核心结构体，简称 “节点数据结构”

非NUMA架构的机器只有一个 pglist_data结构。

```c
typedef struct pglist_data {
/*
 * node_zones 包含**当前节点**的所有内存区域（zone）。
 * 并不是所有的 zone 都一定被初始化或有内存，但这个列表是完整的。
 * 它不仅被当前节点的 node_zonelists 引用，也可能被其它节点的 node_zonelists 引用。
 */
	struct zone node_zones[MAX_NR_ZONES];

/*
 * node_zonelists 包含对所有节点中所有内存区域（zone）的引用。
 * 通常，最前面的 zone 是对本节点的 node_zones 的引用。
 */
	struct zonelist node_zonelists[MAX_ZONELISTS];

//部分省略
} pg_data_t;
```

node_zones 数组包含了不同的内存区域(zone)

### 2.2 zone

用于表示 内存管理中的一个“Zone”（区域） 的核心结构，定义在 include/linux/mmzone.h 中。它是 物理内存分布的一个逻辑划分单位。

```c
struct zone {
	const char		*name;
  struct free_area	free_area[MAX_ORDER];
}
```

free_area 数组保存了不同order(以page为基准，order 0表示1个page； order 1表示2个page， order 2表示4个page，以此类推)下的可用内存。


### 2.3 free_area

用于表示某一阶（order）的空闲页块信息。

```c
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};
```

free_list 数组表示各迁移类型(MIGRATE_UNMOVABLE, MIGRATE_MOVABLE,MIGRATE_RECLAIMABLE等)的空闲页链表，用于 buddy 分配与合并。
nr_free 表示当前阶下的空闲块总数，用于判断是否能满足分配请求。

### 2.4 page

page是linux 内存管理的核心结构。在linux中内存会被分成一个一个页，而page就是用来描述对应的页内存。

它在没有被使用之前，是通过 buddy_list 链接在 free_area的free_list 链表上的。

```c
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/*
	 * Five words (20/40 bytes) are available in this union.
	 * WARNING: bit 0 of the first word is used for PageTail(). That
	 * means the other users of this union MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		struct {	/* Page cache and anonymous pages */
			/**
			 * @lru: Pageout list, eg. active_list protected by
			 * lruvec->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 */
			union {
				struct list_head lru;
				struct {
					void *__filler;
					unsigned int mlock_count;
				};

				/* Or, free page */
				struct list_head buddy_list;    //通过这个链接到free_list链表。
				struct list_head pcp_list;
			};
		};
  };
}；
//部分省略
} _struct_page_alignment;
```

## 3. 内存操作

### 3.1 alloc_pages



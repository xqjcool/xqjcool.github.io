---
title: "An Overview of the Linux Memory Management Framework"
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

alloc_pages申请内存的基本执行路径如下(快速路径)：

```c
alloc_pages --> __alloc_pages --> get_page_from_freelist --> rmqueue --> rmqueue_buddy --> __rmqueue --> __rmqueue_smallest --> get_page_from_free_area
```

路径上各个函数的简单描述如下：

```c
//显示尝试快速路径，如果找不到合适内存，则尝试慢速路径。
struct page *__alloc_pages(gfp_t gfp, unsigned int order, int preferred_nid,
							nodemask_t *nodemask)
{
//部分省略
	page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);	//快速路径
	if (likely(page))
		goto out;
//部分省略
	page = __alloc_pages_slowpath(alloc_gfp, order, &ac);	//慢速路径
//部分省略
}

//通过各种条件找到特定的zone， 尝试在zone里去分配内存
struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
						const struct alloc_context *ac)
{
}

//主要调用rmqueue_buddy函数
struct page *rmqueue(struct zone *preferred_zone, struct zone *zone, unsigned int order,
			gfp_t gfp_flags, unsigned int alloc_flags, int migratetype)
{
}

//主要调用 __rmqueue 申请内存。同时兼顾其他特定条件下的申请。如果申请的内存检查失败，则会重新尝试申请。
struct page *rmqueue_buddy(struct zone *preferred_zone, struct zone *zone, unsigned int order, unsigned int alloc_flags, int migratetype)
{
	struct page *page;
	unsigned long flags;

	do {
		page = NULL;
		spin_lock_irqsave(&zone->lock, flags);
		/*
		 * order-0 request can reach here when the pcplist is skipped
		 * due to non-CMA allocation context. HIGHATOMIC area is
		 * reserved for high-order atomic allocation, so order-0
		 * request should skip it.
		 */
		if (order > 0 && alloc_flags & ALLOC_HARDER)
			page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
		if (!page) {
			page = __rmqueue(zone, order, migratetype, alloc_flags);
			if (!page) {
				spin_unlock_irqrestore(&zone->lock, flags);
				return NULL;
			}
		}
		__mod_zone_freepage_state(zone, -(1 << order),
					  get_pcppage_migratetype(page));
		spin_unlock_irqrestore(&zone->lock, flags);
	} while (check_new_pages(page, order));

	__count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
	zone_statistics(preferred_zone, zone, 1);

	return page;
}

//调用__rmqueue_smallest分配内存。如果失败，则尝试使用其他migratetype来分配。
struct page *__rmqueue(struct zone *zone, unsigned int order, int migratetype, unsigned int alloc_flags)
{
	struct page *page;

	if (IS_ENABLED(CONFIG_CMA)) {
		/*
		 * Balance movable allocations between regular and CMA areas by
		 * allocating from CMA when over half of the zone's free memory
		 * is in the CMA area.
		 */
		if (alloc_flags & ALLOC_CMA &&
		    zone_page_state(zone, NR_FREE_CMA_PAGES) >
		    zone_page_state(zone, NR_FREE_PAGES) / 2) {
			page = __rmqueue_cma_fallback(zone, order);
			if (page)
				return page;
		}
	}
retry:
	page = __rmqueue_smallest(zone, order, migratetype);
	if (unlikely(!page)) {
		if (alloc_flags & ALLOC_CMA)
			page = __rmqueue_cma_fallback(zone, order);

		if (!page && __rmqueue_fallback(zone, order, migratetype,
								alloc_flags))
			goto retry;
	}
	return page;
}

//从当前order开始如果获取不到内存，则增加order继续尝试。
struct page * __rmqueue_smallest(struct zone *zone, unsigned int order, int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		page = get_page_from_free_area(area, migratetype);
		if (!page)
			continue;
		del_page_from_free_list(page, zone, current_order);
		expand(zone, page, order, current_order, migratetype);
		set_pcppage_migratetype(page, migratetype);
		trace_mm_page_alloc_zone_locked(page, order, migratetype,
				pcp_allowed_order(order) &&
				migratetype < MIGRATE_PCPTYPES);
		return page;
	}

	return NULL;
}

//从area的migratetype对应的链表中取空闲内存。如果不存在则返回NULL
struct page *get_page_from_free_area(struct free_area *area,
					    int migratetype)
{
	return list_first_entry_or_null(&area->free_list[migratetype],
					struct page, lru);
}
```



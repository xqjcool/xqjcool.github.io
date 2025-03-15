---
title: "[Memory Leak] Memory Leak Caused by Docker Using cgroup memory"
date: 2021-03-30
---

# 1. 问题现象及分析
## 1.1 问题现象
公司产品（centos 3.10.0-514）需要用频繁启动docker，每个docker的运行时间不固定，一段时间后设备内存不足。

使用内存泄漏分析工具分析发现是slab占用过高
![image](https://github.com/user-attachments/assets/74b3c0d1-f17b-4bbf-9b2d-8cb028a05ac9)

进一步查看发现存在太多的 cgroup memory申请。
![image](https://github.com/user-attachments/assets/9ee25f85-5b50-49c1-b889-f5b91508ce9a)

## 1.2 问题分析
这些都是docker创建时通过 cgroup memory申请的slab，但是docker关闭时，这些slab却没有释放。导致越积越多，将内存消耗殆尽。

经过调查，这是3.x版本内核bug。
请参考
https://support.d2iq.com/s/article/Critical-Issue-KMEM-MSPH-2018-0006
和
https://github.com/torvalds/linux/commit/d6e0b7fa11862433773d986b5f995ffdf47ce672

# 2. 解决办法和尝试
网上推荐了几个方案，相对可操作的有2个。
## 2.1 升级到修复后的版本。
将系统升级到修复后的4.x版本。
不太可行，好多产品已经售出，且升级版本风险太大，太多东西需要重新适配。否掉。

## 2.2 禁用cgroup的kmem属性。
通过修改虚机启动的引导项 grub 中的cgroup.memory=nokmem，让机器启动时直接禁用 cgroup的 kmem 属性。
这个方案操作方便，影响小。于是采用了该方案。


# 3. 方案无效及分析
查到解决方法后，心情大好，本以为问题解决。谁知道增加 cgroup.memory=nokmem 启动项，重启设备后，问题依旧。看起来这个配置并未生效。
## 3.1 确认内核是否支持该选项
因为配置为生效，所以怀疑内核不支持该选项。如何确认呢？
经过分析内核启动项都对应 __setup宏定义，指定相应boot参数对应的处理函数。

```c
/*
 * Only for really core code.  See moduleparam.h for the normal way.
 *
 * Force the alignment so the compiler doesn't space elements of the
 * obs_kernel_param "array" too far apart in .init.setup.
 */
#define __setup_param(str, unique_id, fn, early)			\
	static const char __setup_str_##unique_id[] __initconst	\
		__aligned(1) = str; \
	static struct obs_kernel_param __setup_##unique_id	\
		__used __section(.init.setup)			\
		__attribute__((aligned((sizeof(long)))))	\
		= { __setup_str_##unique_id, fn, early }

#define __setup(str, fn)					\
	__setup_param(str, fn, fn, 0)
```

如 
__setup("console=", console_setup);		//用来分析 console=ttyS0,38400n8 选项
__setup("root=", root_dev_setup);			//用来分析 root=/dev/mapper/cl_cpe-root

我在系统源码中并未找到 cgroup.memory 对应的 __setup定义。

又在linux最新源码中看到  cgroup.memory 的__setup定义确实存在。通过查找发现这是在2016年的一个commit中添加的。

```c
commit f7e1cb6ec51b041335b5ad4dd7aefb37a56d79a6
Author: Johannes Weiner <hannes@cmpxchg.org>
Date:   Thu Jan 14 15:21:29 2016 -0800

    mm: memcontrol: account socket memory in unified hierarchy memory controller
    
    Socket memory can be a significant share of overall memory consumed by
    common workloads.  In order to provide reasonable resource isolation in
    the unified hierarchy, this type of memory needs to be included in the
    tracking/accounting of a cgroup under active memory resource control.
    
    Overhead is only incurred when a non-root control group is created AND
    the memory controller is instructed to track and account the memory
    footprint of that group.  cgroup.memory=nosocket can be specified on the
    boot commandline to override any runtime configuration and forcibly
    exclude socket memory from active memory resource control.
    
    Signed-off-by: Johannes Weiner <hannes@cmpxchg.org>
    Acked-by: David S. Miller <davem@davemloft.net>
    Reviewed-by: Vladimir Davydov <vdavydov@virtuozzo.com>
    Acked-by: Michal Hocko <mhocko@suse.com>
    Signed-off-by: Andrew Morton <akpm@linux-foundation.org>
    Signed-off-by: Linus Torvalds <torvalds@linux-foundation.org>
...//省略部分
@@ -5602,10 +5654,32 @@ bool mem_cgroup_charge_skmem(struct mem_cgroup *memcg, unsigned int nr_pages)
  */
 void mem_cgroup_uncharge_skmem(struct mem_cgroup *memcg, unsigned int nr_pages)
 {
-       page_counter_uncharge(&memcg->tcp_mem.memory_allocated, nr_pages);
+#ifdef CONFIG_MEMCG_KMEM
+       if (!cgroup_subsys_on_dfl(memory_cgrp_subsys)) {
+               page_counter_uncharge(&memcg->tcp_mem.memory_allocated,
+                                     nr_pages);
+               return;
+       }
+#endif
+       page_counter_uncharge(&memcg->memory, nr_pages);
+       css_put_many(&memcg->css, nr_pages);
 }
 
-#endif
+#endif /* CONFIG_INET */
+
+static int __init cgroup_memory(char *s)
+{
+       char *token;
+
+       while ((token = strsep(&s, ",")) != NULL) {
+               if (!*token)
+                       continue;
+               if (!strcmp(token, "nosocket"))
+                       cgroup_memory_nosocket = true;
+       }
+       return 0;
+}
+__setup("cgroup.memory=", cgroup_memory);

```
最终确认当前系统不支持通过 cgroup.memory=nokmem 启动项来关闭cgroup memory的kmem功能。

## 3.2 新的解决方案
查看系统启动log时，发现有这一行提示：

```bash
Aug 17 11:47:47 VM kernel: [    0.000000] allocated 8388608 bytes of page_cgroup
Aug 17 11:47:47 VM kernel: [    0.000000] please try 'cgroup_disable=memory' option if you don't want memory cgroups

```

进一步查看源码，发现可以通过 cgroup_disable=memory 直接禁止cgroup 的 memory功能。
同时也在Kernel-parameters.txt 中找到对该选项的详细描述：
```bash
cgroup_disable= [KNL] Disable a particular controller 
	Format: {name of the controller(s) to disable} {Currently supported controllers - "memory"}
```

既然单独关闭cgroup memory的 kmem不支持，那就关闭掉整个cgroup memory。

修改/boog/grub2/grub.cfg 增加 cgroup_disable=memory 后，重启系统。docker运行正常，没有内存泄漏。




# 太长不看版本
1. 3.x版本cgroup.memory存在bug，docker或其他使用cgroup.memory中kmem功能的应用会造成slab内存泄漏
2. 部分系统可以通过升级到4.x的修复版本来解决，或者通过 cgroup.memory=nokmem 启动项来规避。
3. 部分系统不支持cgroup.memory=nokmem 启动项，可以通过 cgroup_disable=memory 启动项来规避。

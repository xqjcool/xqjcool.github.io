---
title: "[Linux Memory Issue] Investigation of `VmallocUsed` Showing 0 in `meminfo`"
date: 2021-06-08
---

为了便于定位内存问题，使用/proc/meminfo中的一些关键变量值，来计算用户态内存占用，内核态内存占用（包含slab占用和vmalloc占用等），以及黑洞内存占用。(详见<内存泄漏定位思路与方法>)。公司技术同事使用该方法定位了很多项目内存问题。

今天技术同事说有个设备内存占用很高，使用定位方法发现存在隐形内存。我登录设备初步排查了黑洞内存常见的几个怀疑点，没有发现问题。

随后我再次查看meminfo确认，发现了一个奇怪的现象。

```bash
[root@localhost ~]# cat /proc/meminfo 
MemTotal:        1653372 kB
MemFree:          942324 kB
MemAvailable:     778112 kB
Buffers:           46084 kB
Cached:           226908 kB
SwapCached:            0 kB
Active:           261400 kB
Inactive:         118008 kB
//...省略
WritebackTmp:          0 kB
CommitLimit:      826684 kB
Committed_AS:     366212 kB
VmallocTotal:   135290290112 kB
VmallocUsed:           0 kB		//虚拟内存使用值为0
VmallocChunk:          0 kB
Percpu:              944 kB
//...省略
```
VmallocUsed为0kB，这是不可能的。随后查看 vmallocinfo确认存在vmalloc内存申请。

```bash
[root@localhost ~]# cat /proc/vmallocinfo 
0x000000007266083f-0x00000000071a5b85   20480 load_module+0xe4c/0x1e60 pages=4 vmalloc
0x00000000071a5b85-0x000000007ab402cc   20480 load_module+0xe4c/0x1e60 pages=4 vmalloc
0x00000000ff0af38c-0x00000000198b6f2c   20480 load_module+0xe4c/0x1e60 pages=4 vmalloc
0x0000000049ed6fb6-0x00000000064f3656  335872 load_module+0xe4c/0x1e60 pages=81 vmalloc
0x00000000064f3656-0x000000003614d157 5185536 load_module+0xe4c/0x1e60 pages=1265 vmalloc vpages
0x000000003614d157-0x0000000017daddad 2375680 load_module+0xe4c/0x1e60 pages=579 vmalloc vpages
0x000000001bd070ca-0x000000006d25a57e   20480 start_kernel+0x288/0x414 pages=4 vmalloc
0x000000006d25a57e-0x00000000566300b3    8192 of_iomap+0x4c/0xa8 phys=0x0000000001ee1000 ioremap
0x000000003c637984-0x00000000c72ec8ec   20480 start_kernel+0x288/0x414 pages=4 vmalloc
0x00000000c72ec8ec-0x00000000c5c58193    8192 bpf_prog_alloc+0x6c/0x108 pages=1 vmalloc
0x00000000edb49432-0x0000000047024977   20480 start_kernel+0x288/0x414 pages=4 vmalloc
0x0000000047024977-0x000000000a3b860e    8192 __devm_ioremap+0xb8/0x150 phys=0x0000000001ee0000 ioremap
//...省略
```

正是因为vmalloc内存没有统计上，让公式将vmalloc内存统计成隐形内存了。那么为啥统计出的VmallocUsed为0呢？

我看了下系统版本，是CentOS系统 kernel 4.19.26版本。

```bash
[root@localhost ~]# uname -a
Linux cpe 4.19.26-00780-gfdd0789195dd #1 SMP PREEMPT Thu May 27 16:19:59 CST 2021 aarch64 aarch64 aarch64 GNU/Linux
[root@cpe ~]# cat /etc/redhat-release 
CentOS Linux release 7.8.2003 (AltArch)
```

查看了下4.19版本的 meminfo的实现。发现VmallocUsed被hardcode成0 kB了。这是什么鬼......

```c
static int meminfo_proc_show(struct seq_file *m, void *v)
{
	struct sysinfo i;
	unsigned long committed;
	long cached;
	long available;
	unsigned long pages[NR_LRU_LISTS];
	int lru;
//...省略
	show_val_kb(m, "CommitLimit:    ", vm_commit_limit());
	show_val_kb(m, "Committed_AS:   ", committed);
	seq_printf(m, "VmallocTotal:   %8lu kB\n",
		   (unsigned long)VMALLOC_TOTAL >> 10);
	show_val_kb(m, "VmallocUsed:    ", 0ul);	//写死成0
	show_val_kb(m, "VmallocChunk:   ", 0ul);
//...省略
}

```

追溯了一下，是这个commit搞的鬼，他在优化meminfo实现的时候，顺手(有意还是无意？)将VmallocUsed就直接写死为0 kB了。这个commit 在kernel 4.9版本就包含了。

```bash
===============将VmallocUsed写死成0的commit=======
commit e16e2d8e14a14bd87df8482c637dde8f760a8d5f
Author: Joe Perches <joe@perches.com>
Date:   Fri Oct 7 17:02:23 2016 -0700

    meminfo: break apart a very long seq_printf with #ifdefs
    
    Use a specific routine to emit most lines so that the code 
//...省略
+       show_val_kb(m, "VmallocUsed:    ", 0ul);
+       show_val_kb(m, "VmallocChunk:   ", 0ul);
//...省略
====================
```

继续查找，看看该问题是否在新版本修复。发现该问题一直持续到5.3才修复。也就是说 kernel 4.9～5.2之间的版本都包含该问题。

```bash
commit 97105f0ab7b877a8ece2005e214894e93793950c
Author: Roman Gushchin <guro@fb.com>
Date:   Thu Jul 11 21:00:13 2019 -0700
//...省略
-       show_val_kb(m, "VmallocUsed:    ", 0ul);
+       show_val_kb(m, "VmallocUsed:    ", vmalloc_nr_pages());
//...省略
```

使用linux kernel 4.9～5.2版本，需要注意下该问题，避免被meminfo中的VmallocUsed误导。

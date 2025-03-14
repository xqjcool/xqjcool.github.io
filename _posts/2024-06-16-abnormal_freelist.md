---
title: "[CRASH] Abnormal Address Access Caused by Freelist Corruption"
date: 2024-06-16
---

# 问题现象

项目一台设备几天内出现了两次crash，都是异常地址访问导致。

```bash
[66005.261660] BUG: unable to handle page fault for address: ffffff8881575110
```

# 初步分析
拿到coredump后，发现问题出在kmem_cache_cpu的freelist指针上。

```bash
crash> bt
PID: 4685     TASK: ffff88816f2b0000  CPU: 3    COMMAND: "mond"
 #0 [ffffc9000590f818] machine_kexec at ffffffff80284532
 #1 [ffffc9000590f898] __crash_kexec at ffffffff80383b82
 #2 [ffffc9000590f960] oops_end at ffffffff80248731
 #3 [ffffc9000590f980] page_fault_oops at ffffffff8028f0a6
 #4 [ffffc9000590f9a8] xas_create at ffffffff80f5272e
 #5 [ffffc9000590f9b0] brd_do_bvec at ffffffff809c9a60
 #6 [ffffc9000590fa08] kernelmode_fixup_or_oops at ffffffff8028f539
 #7 [ffffc9000590fa78] __bad_area_nosemaphore at ffffffff8028f9e3
 #8 [ffffc9000590faf8] exc_page_fault at ffffffff80f5adfc
 #9 [ffffc9000590fb20] asm_exc_page_fault at ffffffff81000b62
    [exception RIP: __kmem_cache_alloc_node+119]
    RIP: ffffffff80530257  RSP: ffffc9000590fbd0  RFLAGS: 00010246
    RAX: 0000000000000020  RBX: 0000000000000035  RCX: 0000000000000035
    RDX: 0000000001dfcf36  RSI: 0000000000000dc0  RDI: ffff888100041500
    RBP: ffffff88815750f0   R8: ffff888276dad410   R9: 0000000000000035
    R10: 0000000073646134  R11: 0000000004040404  R12: 00000000ffffffff
    R13: ffffffff80641477  R14: ffff888100041500  R15: 0000000000000dc0
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffffc9000590fc00] __kmalloc at ffffffff804c9e77
#11 [ffffc9000590fc40] htree_dirblock_to_tree at ffffffff80641477
#12 [ffffc9000590fcc8] ext4_htree_fill_tree at ffffffff80640f75
#13 [ffffc9000590fdc8] ext4_readdir at ffffffff805f7c32
#14 [ffffc9000590fe88] iterate_dir at ffffffff80560d2b
#15 [ffffc9000590fed0] __x64_sys_getdents64 at ffffffff80561490
#16 [ffffc9000590ff28] do_syscall_64 at ffffffff80f57f28
#17 [ffffc9000590ff50] entry_SYSCALL_64_after_hwframe at ffffffff8100009b

crash> dis -l __kmem_cache_alloc_node+119
/code/FortiWEB/kernel/linux-6.1/mm/slub.c: 374    //获取内存ptr+offset位置保存的next空闲内存指针
0xffffffff80530257 <__kmem_cache_alloc_node+119>:       mov    0x0(%rbp,%rax,1),%rbx

```
通过上面基本能推断出是freelist地址异常导致的这次问题。
为了使大家看的更清晰，也把汇编代码列出，关键过程加备注解释，方便理解：

```bash
crash> dis -r __kmem_cache_alloc_node+119
0xffffffff805301e0 <__kmem_cache_alloc_node>:   nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff805301e5 <__kmem_cache_alloc_node+5>: push   %rbp
0xffffffff805301e6 <__kmem_cache_alloc_node+6>: push   %r15
0xffffffff805301e8 <__kmem_cache_alloc_node+8>: push   %r14
0xffffffff805301ea <__kmem_cache_alloc_node+10>:        push   %r13
0xffffffff805301ec <__kmem_cache_alloc_node+12>:        push   %r12
0xffffffff805301ee <__kmem_cache_alloc_node+14>:        push   %rbx
0xffffffff805301ef <__kmem_cache_alloc_node+15>:        test   %rdi,%rdi
0xffffffff805301f2 <__kmem_cache_alloc_node+18>:        je     0xffffffff805302ae <__kmem_cache_alloc_node+206>
0xffffffff805301f8 <__kmem_cache_alloc_node+24>:        mov    %r8,%r13
0xffffffff805301fb <__kmem_cache_alloc_node+27>:        mov    %rcx,%r9
0xffffffff805301fe <__kmem_cache_alloc_node+30>:        mov    %edx,%r12d
0xffffffff80530201 <__kmem_cache_alloc_node+33>:        mov    %esi,%r15d
0xffffffff80530204 <__kmem_cache_alloc_node+36>:        mov    %rdi,%r14  //kmem_cache *s
0xffffffff80530207 <__kmem_cache_alloc_node+39>:        cmpl   $0x0,0x22f33f2(%rip)        # 0xffffffff82823600 <kfence_allocation_key>
0xffffffff8053020e <__kmem_cache_alloc_node+46>:        jle    0xffffffff8053021d <__kmem_cache_alloc_node+61>
0xffffffff80530210 <__kmem_cache_alloc_node+48>:        cmpl   $0x0,0x1a4da31(%rip)        # 0xffffffff81f7dc48 <kfence_allocation_gate>
0xffffffff80530217 <__kmem_cache_alloc_node+55>:        je     0xffffffff80530323 <__kmem_cache_alloc_node+323>
0xffffffff8053021d <__kmem_cache_alloc_node+61>:        mov    (%r14),%r8  //kmem_cache_cpu *cpu_slab
0xffffffff80530220 <__kmem_cache_alloc_node+64>:        add    %gs:0x7fae5360(%rip),%r8        # 0x15588
0xffffffff80530228 <__kmem_cache_alloc_node+72>:        mov    0x8(%r8),%rdx
0xffffffff8053022c <__kmem_cache_alloc_node+76>:        mov    (%r8),%rbp  //void **freelist
0xffffffff8053022f <__kmem_cache_alloc_node+79>:        test   %rbp,%rbp
0xffffffff80530232 <__kmem_cache_alloc_node+82>:        je     0xffffffff805302be <__kmem_cache_alloc_node+222>
0xffffffff80530238 <__kmem_cache_alloc_node+88>:        mov    0x10(%r8),%rax
0xffffffff8053023c <__kmem_cache_alloc_node+92>:        test   %rax,%rax
0xffffffff8053023f <__kmem_cache_alloc_node+95>:        je     0xffffffff805302be <__kmem_cache_alloc_node+222>
0xffffffff80530241 <__kmem_cache_alloc_node+97>:        cmp    $0xffffffff,%r12d
0xffffffff80530245 <__kmem_cache_alloc_node+101>:       je     0xffffffff80530253 <__kmem_cache_alloc_node+115>
0xffffffff80530247 <__kmem_cache_alloc_node+103>:       mov    (%rax),%rax
0xffffffff8053024a <__kmem_cache_alloc_node+106>:       shr    $0x3a,%rax
0xffffffff8053024e <__kmem_cache_alloc_node+110>:       cmp    %r12d,%eax
0xffffffff80530251 <__kmem_cache_alloc_node+113>:       jne    0xffffffff805302be <__kmem_cache_alloc_node+222>
0xffffffff80530253 <__kmem_cache_alloc_node+115>:       mov    0x28(%r14),%eax  //offset
0xffffffff80530257 <__kmem_cache_alloc_node+119>:       mov    0x0(%rbp,%rax,1),%rbx  //*(freelist + offset) 获取内存ptr+offset位置保存的next空闲内存指针

```

R14=RDI=(struct kmem_cache *)ffff888100041500
RAX=offset=0x20
RBP=freelist=ffffff88815750f0   
为了严谨，通过kmem_cache确认一下。

```bash
struct kmem_cache {
  cpu_slab = 0x2d410,
  flags = 0x40001000,
  min_partial = 0x5,
  size = 0x40,
  object_size = 0x40,
  reciprocal_size = {
    m = 0x1,
    sh1 = 0x1,
    sh2 = 0x5
  },
  offset = 0x20,	//RAX
  cpu_partial = 0x78,
  cpu_partial_slabs = 0x4,
  oo = {
    x = 0x40
  },
  min = {
    x = 0x40
  },
  allocflags = 0x0,
  refcount = 0x1,
  ctor = 0x0,
  inuse = 0x40,
  align = 0x40,
  red_left_pad = 0x0,
  name = 0xffffffff81582d0a "kmalloc-64",	//通用的64字节的slab分配器


crash> kmem_cache_cpu 0x2d410:3		//在问题CPU3上的 cpu_slab
[3]: ffff888276dad410
struct kmem_cache_cpu {
  freelist = 0xffffff88815750f0,	//RBP 问题freelist
  tid = 31444790,
  slab = 0xffffea00055d43c0,
  partial = 0xffffea000400fac0,
  lock = {<No data fields>}
}

```
也就是说 通用slab分配器 kmalloc-64在CPU3上的freelist异常，导致新申请内存时因为无法访问freelist+offset去获取下一个空闲object内存指针，导致了系统crash。 

# 继续深入
接下来到比较头疼的地方了。freelist是怎么异常的？

是内核或者模块中有kfree(ptr=0xffffff88815750f0)将这个指针放回到freelist上的么？
答案是否定的！因为kfree释放地址时，slab会将原 freelist指针写到 ptr+kmem_cache.offset位置。而ptr=0xffffff88815750f0 是不可访问的地址，在kfree时就会出发地址访问异常。

我们再次审视 0xffffff88815750f0 这个异常地址，发现它比较怪异。它更像是一个正常地址的偏移。

从kmem_cache_cpu.slab我们可以知道slab对应的内存范围 ffff88815750f000 ~ ffff88815750ffff。

```bash
crash> kmem 0xffffea00055d43c0
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
kmem: kmalloc-64: slab: ffffea00055d43c0 invalid freepointer: ffffff8881575110
ffff888100041500       64       4063      4608     72     4k  kmalloc-64
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
kmem: kmalloc-64: slab: ffffea00055d43c0 invalid freepointer: ffffff8881575110
  ffffea00055d43c0  ffff88815750f000     0     64         63     1
  FREE / [ALLOCATED]
kmem: kmalloc-64: slab: ffffea00055d43c0 invalid freepointer: ffffff8881575110

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00055d43c0 15750f000 ffff888100041500        0  1 200000000000200 slab

```


**slab对应地址**    ff ff 88 81 57 50 f0 00
**异常freelist**    ff ff ff 88 81 57 50 f0

感觉像是 一个正常地址存在内存中，读取时偏移了一个字节。
00 f0 50 57 81 88  ff  ff  ff 
 0   1   2   3   4   5  6  7  8

从0-7 就是 ffff88815750f000 ， 从1到8 就是 ffffff88815750f0。

# 新的发现
异常内存地址的特点，基本指向了use_after_free，也就是说内存归还给slab分配器后，又对内存进行操作，导致异常。
但是如何推演出问题场景，还没有好的思路。另外kmalloc-64是通用的分配器，可能各种地方都在使用，一时不好去怀疑是哪个地方可能出了问题。

由于工作上同时有多个bug要处理，暂时搁置了下。然后QA报告该设备又出现一次crash，现象有些不同。

```bash
crash> bt
PID: 31715    TASK: ffff888106ef2e80  CPU: 2    COMMAND: "mond"
 #0 [ffffc90004f1f908] machine_kexec at ffffffff80284532
 #1 [ffffc90004f1f980] __crash_kexec at ffffffff80383b82
 #2 [ffffc90004f1fa48] oops_end at ffffffff80248731
 #3 [ffffc90004f1fa68] exc_general_protection at ffffffff80f588dd
 #4 [ffffc90004f1fb90] asm_exc_general_protection at ffffffff81000aa2
    [exception RIP: rb_insert_color+75]
    RIP: ffffffff80f4638b  RSP: ffffc90004f1fc40  RFLAGS: 00010246
    RAX: ffff8881018bfdd0  RBX: 0000000000000005  RCX: ff88810242fd093a
    RDX: ffff8881274abe48  RSI: ffff88810242fa80  RDI: ffff888141437f88
    RBP: ffff88810242f0c8   R8: ffff88810242fd88   R9: ffff8881274ab180
    R10: 0000000070747962  R11: 0000000005050505  R12: ffff8881274ab180
    R13: 00000000454ccceb  R14: 000000000d8620ac  R15: ffff88817a984a60
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #5 [ffffc90004f1fc40] htree_dirblock_to_tree at ffffffff80641531
 #6 [ffffc90004f1fcc8] ext4_htree_fill_tree at ffffffff80640f75
 #7 [ffffc90004f1fdc8] ext4_readdir at ffffffff805f7c32
 #8 [ffffc90004f1fe88] iterate_dir at ffffffff80560d2b
 #9 [ffffc90004f1fed0] __x64_sys_getdents64 at ffffffff80561490
#10 [ffffc90004f1ff28] do_syscall_64 at ffffffff80f57f28
#11 [ffffc90004f1ff50] entry_SYSCALL_64_after_hwframe at ffffffff8100009b

```

这次是出在红黑树的插入时。巧的是错误地址   ff88810242fd093a 也像是读取发生偏移导致。

```bash
crash> dis -l rb_insert_color+75
/code/FortiWEB/kernel/linux-6.1/lib/rbtree.c: 115
0xffffffff80f4638b <rb_insert_color+75>:        mov    0x8(%rcx),%rdx

crash> dis -r rb_insert_color+75
0xffffffff80f46340 <rb_insert_color>:   mov    (%rdi),%r8
0xffffffff80f46343 <rb_insert_color+3>: test   %r8,%r8
0xffffffff80f46346 <rb_insert_color+6>: jne    0xffffffff80f4637f <rb_insert_color+63>
0xffffffff80f46348 <rb_insert_color+8>: mov    %rdi,%rcx
0xffffffff80f4634b <rb_insert_color+11>:        movq   $0x1,(%rcx)
0xffffffff80f46352 <rb_insert_color+18>:        ret    
0xffffffff80f46353 <rb_insert_color+19>:        nopw   %cs:0x0(%rax,%rax,1)
0xffffffff80f4635d <rb_insert_color+29>:        nopl   (%rax)
0xffffffff80f46360 <rb_insert_color+32>:        mov    %rcx,%rax
0xffffffff80f46363 <rb_insert_color+35>:        or     $0x1,%rax
0xffffffff80f46367 <rb_insert_color+39>:        mov    %rax,(%rdx)
0xffffffff80f4636a <rb_insert_color+42>:        mov    %rax,(%r8)
0xffffffff80f4636d <rb_insert_color+45>:        mov    (%rcx),%r8
0xffffffff80f46370 <rb_insert_color+48>:        and    $0xfffffffffffffffc,%r8
0xffffffff80f46374 <rb_insert_color+52>:        mov    %r8,(%rcx)
0xffffffff80f46377 <rb_insert_color+55>:        mov    %rcx,%rdi
0xffffffff80f4637a <rb_insert_color+58>:        test   %r8,%r8
0xffffffff80f4637d <rb_insert_color+61>:        je     0xffffffff80f4634b <rb_insert_color+11>
0xffffffff80f4637f <rb_insert_color+63>:        mov    (%r8),%rcx
0xffffffff80f46382 <rb_insert_color+66>:        test   $0x1,%cl
0xffffffff80f46385 <rb_insert_color+69>:        jne    0xffffffff80f4649f <rb_insert_color+351>
0xffffffff80f4638b <rb_insert_color+75>:        mov    0x8(%rcx),%rdx

```

通过反汇编，找到红黑树的根节点。

```bash
crash> dir_private_info.root 0xffff88810242fa80 
  root = {
    rb_node = 0xffff88810242f688
  },

crash> fname -x ffff88810242fd80
struct fname {
  hash = 0xb2f52600,
  minor_hash = 0x32115811,
  rb_hash = {
    __rb_parent_color = 0xff88810242fd093a,		//这里就是异常发生的地址
    rb_right = 0xffff888141437f88,
    rb_left = 0xff88810242f0c8ff				//异常
  },
  next = 0xff,   //明显异常
  inode = 0x7a5900,
  name_len = 0x0,
  file_type = 0x5,
  name = 0xffff88810242fdae "\003ttyac"	//明显异常
}

```
然后试着将ffff88810242fd80 偏移1个字节，再按fname结构去读取

```bash
crash> fname -x ffff88810242fd81
struct fname {
  hash = 0x11b2f526,
  minor_hash = 0x3a321158,
  rb_hash = {
    __rb_parent_color = 0x88ff88810242fd09,
    rb_right = 0xffffff888141437f,
    rb_left = 0xffff88810242f0c8
  },
  next = 0x0,
  inode = 0x7a59,
  name_len = 0x5,
  file_type = 0x3,
  name = 0xffff88810242fdaf "ttyac"
}

```

推断是申请fname时返回的是 ffff88810242fd81， 而在红黑树操作和旋转时，又以ffff88810242fd80去访问（最低位的1bit被当作红黑树的着色而或掉了）。

kmalloc时返回ffff88810242fd81，表明有use_after_free将 freelist+offset里的地址ffff88810242fd80改为了ffff88810242fd81。

要想满足这个并不容易，因为普通的覆写会整个破坏。像这种只改一个bit的，一般有两种情况：
1. atomic_inc 
2. set_bit

也就是说存在一个结构，偏移0x20的位置是一个atomic或flags变量。当内存free后，这里保存的时next free object的地址指针。此时对这个结构加偏移位置内存的atomic_inc或set_bit，导致next free object地址指针的改变。此时kmalloc申请到的内存指针就会是正常指针ptr偏移1byte。

# 沙盘推演
kmallo获取地址偏移1byte的原因理清楚了。那么最早的 这个异常freelist = ffffff88815750f0 是怎么来的呢？

聪明的读者可能一下子就明白了。不明白的没关系，我也是又想了好久才弄清晰的。
答案就是next free的next free。

```bash
freelist--|                   |-----正确next----|     正确地址    
          v                   |                v        v
          ---------------A---------------       -------------------B---------------------
          |          |偏移1个byte|       |      |00|    |00|f0|50|57|81|88|ff|ff|ff|      |
          -------------------------------       -----------------------------------------
                              |                   ^         ^
                              |-----错误next-------|       错误地址

```
正常情况下freelist+offset位置存储着下一个free object的地址, 而free object+offset位置存储着下下一个free object的地址。
1. 异常use_after_free, 修改了object A内存的 offset位置，导致此位置存储的next指针偏移。
2. kmalloc申请内存，返回object A的地址。此时freelist=*(freelist + offset)位置得到错误的next object(向右偏移了一个byte)。
3. kmalloc申请内存，返回object B+1的地址。一般为freelist向右偏移了1个byte，所以读取到的地址就会变成ffffff88815750f0 。

# 寻找元凶
思路通透以后，就是寻找元凶的时刻了。
kmalloc-64太通用了，不像其他特定的slab分配器，从代码上是不好定位的。
只能通过查看内存进行分析，好在这个分配器一个slab page包含64个object，不算很多。

比较幸运的是object中都留有有效信息，迅速帮我们找到内存的使用(或曾经使用)者。

```bash
crash> rd -s ffff88810bac50c0 8
ffff88810bac50c0:  0000000000000000 broad_proc_cleanup+10752 
ffff88810bac50d0:  ffff888101db68e8 ffff888101db68e8 
ffff88810bac50e0:  0000000000000001 0000000000000000 
ffff88810bac50f0:  0000000000000000 0000000000000000 
//通过broad_proc_cleanup 找到这是个 team_item
crash> groups_item ffff88810bac50c0(4)
struct groups_item {
  hnode = {
    next = 0x0,
    pprev = 0xffffffffa03b1130 <borad_proc_cleanup+10752>
  },
  glist = {
    next = 0xffff888101db68e8,
    prev = 0xffff888101db68e8
  },
  list_account = {
    counter = 1          //atomic_t变量
  }
}

```
这是一个业务模块，巧的是 list_account就是个atomic变量
基本上可以怀疑是这个业务模块对结构的user_after_free导致的。
为了全面，干脆把剩下的也看了。

```bash
crash> rd -s ffff88810bac5240 8
ffff88810bac5240:  0000000000000000 ffffc900098dc000 
ffff88810bac5250:  0000000000005000 0000000000000002 
ffff88810bac5260:  ffff8881457d6de0 0000000400000000 
ffff88810bac5270:  0000000000000000 copy_process+395 

//通过copy_process找到这是个 vm_struct
crash> vm_struct ffff88810bac5240(10)
struct vm_struct {
  next = 0x0,
  addr = 0xffffc900098dc000,
  size = 20480,
  flags = 2,
  pages = 0xffff8881457d6de0,
  page_order = 0,
  nr_pages = 4,
  phys_addr = 0,
  caller = 0xffffffff8029cdab <copy_process+395>
}

```
经过筛选，这个slab page 只有 fname，vm_struct和team_item在使用。
fname和vm_struct是内核代码， 出问题概率非常低，基本就是 业务模块了。

根据之前推导，应该是业务模块在释放 team_item后，又对其进行了 atomic_inc操作导致。

# 分析代码
找到元凶后，就剩下分析模块代码，找到出问题点。
代码问题比较隐蔽，初看没有问题。再仔细分析时，发现存在并发问题。
读写锁对事务的保护不够完善，查找和操作并没有完全保护起来，而是分成两部分保护了。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/784c192118a6249a9ca351c5d283f308.png)
# 后记
因为是总结，所以条理相对清晰有序。实际分析这个问题是遇到不少困难，也走了一些弯路。一度有点沮丧，感觉找不到思路。还好最终找到问题原因。
其中也有一些幸运的地方，比如内存中留下了蛛丝马迹，帮助了解内存用途，如果没有这些，定位难度会大大提高。

---
title: "[memory leak]Objects remaining in slab_test on kmem_cache_close()"
date: 2020-08-01
---
公司产品在重启服务时，log里有如下Call Trace。

```
Jul 29 09:27:39 cpe kernel: [  997.113469] =============================================================================
Jul 29 09:27:39 cpe kernel: [  997.113474] BUG slab_test (Tainted: G           OE  ------------  ): Objects remaining in slab_test on kmem_cache_close()
Jul 29 09:27:39 cpe kernel: [  997.113477] -----------------------------------------------------------------------------
Jul 29 09:27:39 cpe kernel: [  997.113477] 
Jul 29 09:27:39 cpe kernel: [  997.113480] Disabling lock debugging due to kernel taint
Jul 29 09:27:39 cpe kernel: [  997.113484] INFO: Slab 0xffffea0004d9cbc0 objects=32 used=1 fp=0xffff88013672ff80 flags=0x2fffff00000080
Jul 29 09:27:39 cpe kernel: [  997.113489] CPU: 2 PID: 15480 Comm: rmmod Tainted: G    B      OE  ------------   3.10.0-514.26.2.el7.x86_64 #1
Jul 29 09:27:39 cpe kernel: [  997.113492] Hardware name: AEWIN Net Work SCB-1615/Aptio CRB, BIOS 5.6.5 11/12/2018
Jul 29 09:27:39 cpe kernel: [  997.113495]  ffffea0004d9cbc0 000000000887defe ffff880094d1bd10 ffffffff81687133
Jul 29 09:27:39 cpe kernel: [  997.113502]  ffff880094d1bde8 ffffffff811da714 ffff880100000020 ffff880094d1bdf8
Jul 29 09:27:39 cpe kernel: [  997.113507]  ffff880094d1bda8 656a624f029c5ac0 616d657220737463 6e6920676e696e69
Jul 29 09:27:39 cpe kernel: [  997.113512] Call Trace:
Jul 29 09:27:39 cpe kernel: [  997.113521]  [<ffffffff81687133>] dump_stack+0x19/0x1b
Jul 29 09:27:39 cpe kernel: [  997.113529]  [<ffffffff811da714>] slab_err+0xb4/0xe0
Jul 29 09:27:39 cpe kernel: [  997.113535]  [<ffffffff8118a200>] ? alloc_pages_exact+0x10/0x40
Jul 29 09:27:39 cpe kernel: [  997.113540]  [<ffffffff811db0bc>] ? __free_slab+0xdc/0x1f0
Jul 29 09:27:39 cpe kernel: [  997.113545]  [<ffffffff811dc150>] ? kmem_cache_alloc_bulk+0x140/0x140
Jul 29 09:27:39 cpe kernel: [  997.113550]  [<ffffffff811dda13>] ? __kmalloc+0x1f3/0x240
Jul 29 09:27:39 cpe kernel: [  997.113554]  [<ffffffff811e00eb>] ? kmem_cache_close+0x12b/0x2f0
Jul 29 09:27:39 cpe kernel: [  997.113558]  [<ffffffff811e010c>] kmem_cache_close+0x14c/0x2f0
Jul 29 09:27:39 cpe kernel: [  997.113563]  [<ffffffff811e02c4>] __kmem_cache_shutdown+0x14/0x80
Jul 29 09:27:39 cpe kernel: [  997.113569]  [<ffffffff811a5e14>] kmem_cache_destroy+0x44/0xf0
Jul 29 09:27:39 cpe kernel: [  997.113605]  [<ffffffffa07ddd21>] _TEST_PoolDestroy_NL+0x38/0x3a [lwconn]
Jul 29 09:27:39 cpe kernel: [  997.113636]  [<ffffffffa07b380c>] TEST_PoolDestroy+0x8c/0xa0 [lwconn]
Jul 29 09:27:39 cpe kernel: [  997.113662]  [<ffffffffa0751f30>] TEST_RuleExit+0x60/0x80 [lwconn]
Jul 29 09:27:39 cpe kernel: [  997.113686]  [<ffffffffa074001a>] TEST_Exit+0x5a/0x70 [lwconn]
Jul 29 09:27:39 cpe kernel: [  997.113708]  [<ffffffffa073706f>] cleanup_module+0x2f/0xd0 [lwconn]
Jul 29 09:27:39 cpe kernel: [  997.113714]  [<ffffffff810fe3db>] SyS_delete_module+0x16b/0x2d0
Jul 29 09:27:39 cpe kernel: [  997.113720]  [<ffffffff81697809>] system_call_fastpath+0x16/0x1b
Jul 29 09:27:39 cpe kernel: [  997.113726] INFO: Object 0xffff88013672f100 @offset=256
```

很明显 kmem_cache slab_test中有object没有释放，然后在销毁slab_test时，会打印该Call Trace。而该 kmem_cache slab_test并不会释放，造成了内存泄漏。
业务重新启动后，会再次申请 kmem_cache slab_test。于是我们在/proc/slabinfo中会看到2个  kmem_cache slab_test。

```
[root@cpe ~]# cat /proc/slabinfo |grep slab_test
slab_test    320    320    128   32    1 : tunables    0    0    0 : slabdata     10     10      0
slab_test      1     32    128   32    1 : tunables    0    0    0 : slabdata      1      1      0
```
起初怀疑是卸载模块时，相关object因为代码bug没有释放完全。分析相关代码，发现处理逻辑无误，确实遍历链表将所有object都释放了。

而这个现象后续一致没再出现，也没找到复现条件。导致陷入僵局。

有天在思考中突然想，为何不把未释放的object内存dump出来？还原出数据结构来分析问题，说不定能有新的进展。

于是  echo c > /proc/sysrq-trigger 强制系统crash，产生core文件。使用crash工具打开。

```
crash> kmem -s |grep slab_test
ffff8800517ea300 slab_test         84        267       320     10     4k
ffff8801394bca00 slab_test         84          1        32      1     4k
```

通过kmem_cache 获取node地址，再得到node

```
crash> struct kmem_cache.node ffff8801394bca00
  node = {0xffff8801394bd280, ...//省略部分...}
crash> struct kmem_cache_node 0xffff8801394bd280
struct kmem_cache_node {
  list_lock = {
    {
      rlock = {
        raw_lock = {
          {
            head_tail = 33161722, 
            tickets = {
              head = 506, 
              tail = 506
            }
          }
        }
      }
    }
  }, 
  nr_partial = 1, 
  partial = {
    next = 0xffffea0004d9cbe0,         //挂在链表上的page
    prev = 0xffffea0004d9cbe0
  }, 
  nr_slabs = {
    counter = 1
  }, 
  total_objects = {
    counter = 32
  }, 
  full = {
    next = 0xffff8801394bd2b0, 
    prev = 0xffff8801394bd2b0
  }
}
```
page->lru的偏移是0x20， 所以我们可以得到page的指针为0xffffea0004d9cbc0。

```
crash> kmem 0xffffea0004d9cbc0
CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE
ffff8800517ea300 slab_test         84        267       320     10     4k
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea0004d9cbc0  ffff88013672f000     0     32          1    31
  FREE / [ALLOCATED]
   ffff88013672f000  
   ffff88013672f080  
  [ffff88013672f100]				//这是未释放的object地址
   ffff88013672f180  
   ffff88013672f200  
//...省略部分
   ffff88013672ff00  
   ffff88013672ff80  

      PAGE       PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea0004d9cbc0 13672f000                0 ffff88013672ff80  1 2fffff00000080 slab
```
找到未释放object地址后，我们就可以读出其内容了。

```
crash> rd ffff88013672f100 12
ffff88013672f100:  000000004a41434b 0000000000000054   KCAJ....T.......
ffff88013672f110:  0000000000000000 0000000000000000   ................
ffff88013672f120:  ffff88013672f120 ffff88013672f120    .r6.... .r6....		//这是一个list_head，从值上看，可以指导是个刚初始化的。
ffff88013672f130:  000000000000000a ffff880036740800   ..........t6....
ffff88013672f140:  0000000000000000 0000000000000000   ................
ffff88013672f150:  0000000058494e47 0000000000000000   GNIX............
```
对比object的结构定义，发现结构list_head刚初始化，还未挂载到相应链表上。从这里突破，确定问题出在“object申请”和“插入到链表操作”之间。范围大大缩小，查看相关代码，发现在“插入到链表”之前，有个操作失败后，直接返回，没有对应的释放object操作，最终问题原因找到。

后记：
我在回溯定位过程时，发现走了点弯路，我是通过slab-->kmem_cache-->kmem_cache_node-->page-->object指针。实际上log中的Call Trace已经打印了问题object的指针。我们可以直接读取该Object地址。 :-)

> Jul 29 09:27:39 cpe kernel: [  997.113726] INFO: Object 0xffff88013672f100 @offset=256

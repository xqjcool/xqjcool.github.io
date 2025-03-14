---
title: "[Memory Leak] Debugging process for kmalloc-128 slab memory leak."
date: 2022-03-17
---

# 问题现象

前几天，技术团队告知，线上几台ARM CentOS设备内存占用高，比其他相同设备占用高约200M。
查看 free，meminfo和slabinfo后，怀疑是slab kmalloc-128存在内存泄漏。

# 进展受阻
我们发现正常设备和内存占用高设备的 kmalloc-128 占用已经超过177万，粗算下来200多M，刚好是多占用的内存大小。

```bash
//异常设备的slab
kmalloc-128       1767698 1770688    128   32    1 : tunables  120   60    8 : slabdata  55334  55334     60

//正常设备的slab
kmalloc-128        22726  24096    128   32    1 : tunables  120   60    8 : slabdata    753    753      0

```

但这个内存被谁占用了呢？
先是根据《内存泄露定位思路与方法》大概过了一下之前遇到过的泄漏情况，发现都不是。

另外该设备没有打开slab_nomerge开关，导致一些大小接近的slab会被merge成一个slab。也就是说 kmalloc-128 占用高也有可能是别的大小接近的slab泄漏导致。

搜索问题历史发现有个slab的大小占用为128的结构，在开启特定配置情况下，会出现内存泄漏。当时一阵欣喜，以为破案了。哪知和技术人员确认，该设备并没有配置该功能，所以不是该问题导致。

思路一度受阻。

# 现网排查
分析手段受阻，那就在实际环境上操作进行排查。一般情况下是不允许的，幸运的是这个环境是主备环境。我们操作备机不会影响客户业务，所以得到了客户的允许。

第一步是停止和启动主备服务，内存无变化。
第二步是停止和启动业务模块，内存无变化。

两个操作做完后，内存无明显变化，所以也反向确认了非正常使用内存(正常使用在关闭服务和模块时能够释放)。但这些不足以定位到泄露原因。

# 分析coredump
对于x86系统，我们可以使用crash工具实时分析操作系统内存，查看对应slab内存内容来进一步定位问题。不幸的是，问题设备时arm centos系统，不支持crash实时调试。只能手动产生core文件来分析了。

好在备机crash重启不影响客户业务，得到允许后，执行 

> echo c > /proc/sysrq-trigger

手动触发系统crash，产生coredump文件。

使用crash工具加载后，将kmalloc-128 的slab entry打印出来。

```bash
crash> kmem -S kmalloc-128
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffff80007b000000      128    1723323   1723968  53874     4k  kmalloc-128
SLAB              MEMORY            TOTAL  ALLOCATED  FREE
ffff7e0000e96c00  ffff80003a5b0000     32         22    10
FREE / [ALLOCATED]
  [ffff80003a5b0000]
  [ffff80003a5b0080]
  [ffff80003a5b0100]
   ffff80003a5b0180  (cpu 1 cache)
   ffff80003a5b0200
  [ffff80003a5b0280]
  [ffff80003a5b0300]
  [ffff80003a5b0380]
   ffff80003a5b0400
  [ffff80003a5b0480]
  [ffff80003a5b0500]
  [ffff80003a5b0580]
  [ffff80003a5b0600]
   ffff80003a5b0680  (cpu 1 cache)
  [ffff80003a5b0700]
   ffff80003a5b0780
  [ffff80003a5b0800]
  [ffff80003a5b0880]
   ffff80003a5b0900
   ffff80003a5b0980  (cpu 1 cache)
  [ffff80003a5b0a00]
  [ffff80003a5b0a80]
  [ffff80003a5b0b00]
  [ffff80003a5b0b80]
  [ffff80003a5b0c00]
  [ffff80003a5b0c80]
  [ffff80003a5b0d00]
  [ffff80003a5b0d80]
   ffff80003a5b0e00
  [ffff80003a5b0e80]
   ffff80003a5b0f00
   ffff80003a5b0f80
SLAB              MEMORY            TOTAL  ALLOCATED  FREE
ffff7e0000eb1700  ffff80003ac5c000     32         19    13
FREE / [ALLOCATED]
   ffff80003ac5c000
  [ffff80003ac5c080]
  [ffff80003ac5c100]
  [ffff80003ac5c180]
   ffff80003ac5c200
  [ffff80003ac5c280]
  [ffff80003ac5c300]
   ffff80003ac5c380
   ffff80003ac5c400
  [ffff80003ac5c480]
   ffff80003ac5c500
  [ffff80003ac5c580]
   ffff80003ac5c600  (cpu 3 cache)
  [ffff80003ac5c680]
   ffff80003ac5c700
   ffff80003ac5c780  (cpu 0 cache)
  [ffff80003ac5c800]
   ffff80003ac5c880
  [ffff80003ac5c900]
  [ffff80003ac5c980]
  [ffff80003ac5ca00]
   ffff80003ac5ca80  (cpu 0 cache)
  [ffff80003ac5cb00]
  [ffff80003ac5cb80]
   ffff80003ac5cc00
   ffff80003ac5cc80
   ffff80003ac5cd00  (cpu 3 cache)
  [ffff80003ac5cd80]
  [ffff80003ac5ce00]
  [ffff80003ac5ce80]
  [ffff80003ac5cf00]
  [ffff80003ac5cf80]

```

随便挑了几个，看了一下，没有什么有用信息。继续查看，发现一部分slab entry内容一致，且像是代码地址。

```bash
crash> rd ffff80003a5b0000 16
ffff80003a5b0000:  ffff00000824edd8 ffff00000824ee00   ..$.......$.....
ffff80003a5b0010:  ffff00000824ede8 ffff000001415218   ..$......RA.....
ffff80003a5b0020:  0000000000000040 0000000000010b88   @...............
ffff80003a5b0030:  0038004000000000 001d001e00400009   ....@.8...@.....
ffff80003a5b0040:  00010102464c457f 0000000000000000   .ELF............
ffff80003a5b0050:  0000000100b70003 00000000000010e0   ................
ffff80003a5b0060:  0000000000000040 0000000000026d78   @.......xm......
ffff80003a5b0070:  0038004000000000 001a001b00400007   ....@.8...@.....
crash> rd ffff80003a5b0080 16
ffff80003a5b0080:  0000000000000000 ffff00001d045000   .........P......
ffff80003a5b0090:  0000000000022000 0000000000000002   . ..............
ffff80003a5b00a0:  ffff800061795e00 0000000000000021   .^ya....!.......
ffff80003a5b00b0:  0000000000000000 ffff0000014ab824   ........$.J.....
ffff80003a5b00c0:  0000000000000000 0000000000000000   ................
ffff80003a5b00d0:  0000000000000000 0000000000000000   ................
ffff80003a5b00e0:  0000000000000000 0000000000000000   ................
ffff80003a5b00f0:  0000000000000000 0000000000000000   ................

crash> rd ffff80003a5b0100 16
ffff80003a5b0100:  ffff00000824edd8 ffff00000824ee00   ..$.......$.....
ffff80003a5b0110:  ffff00000824ede8 ffff000001415218   ..$......RA.....
ffff80003a5b0120:  0000000000000040 0000000000010b88   @...............
ffff80003a5b0130:  0038004000000000 001d001e00400009   ....@.8...@.....
ffff80003a5b0140:  00010102464c457f 0000000000000000   .ELF............
ffff80003a5b0150:  0000000100b70003 00000000000010e0   ................
ffff80003a5b0160:  0000000000000040 0000000000026d78   @.......xm......
ffff80003a5b0170:  0038004000000000 001a001b00400007   ....@.8...@.....
crash> 
crash> rd ffff80003a5b0000 16
ffff80003a5b0000:  ffff00000824edd8 ffff00000824ee00   ..$.......$.....
ffff80003a5b0010:  ffff00000824ede8 ffff000001415218   ..$......RA.....
ffff80003a5b0020:  0000000000000040 0000000000010b88   @...............
ffff80003a5b0030:  0038004000000000 001d001e00400009   ....@.8...@.....
ffff80003a5b0040:  00010102464c457f 0000000000000000   .ELF............
ffff80003a5b0050:  0000000100b70003 00000000000010e0   ................
ffff80003a5b0060:  0000000000000040 0000000000026d78   @.......xm......
ffff80003a5b0070:  0038004000000000 001a001b00400007   ....@.8...@.....
crash> rd ffff80003a5b0080 16
ffff80003a5b0080:  0000000000000000 ffff00001d045000   .........P......
ffff80003a5b0090:  0000000000022000 0000000000000002   . ..............
ffff80003a5b00a0:  ffff800061795e00 0000000000000021   .^ya....!.......
ffff80003a5b00b0:  0000000000000000 ffff0000014ab824   ........$.J.....
ffff80003a5b00c0:  0000000000000000 0000000000000000   ................
ffff80003a5b00d0:  0000000000000000 0000000000000000   ................
ffff80003a5b00e0:  0000000000000000 0000000000000000   ................
ffff80003a5b00f0:  0000000000000000 0000000000000000   ................
crash> rd ffff80003a5b0100 16
ffff80003a5b0100:  ffff00000824edd8 ffff00000824ee00   ..$.......$.....
ffff80003a5b0110:  ffff00000824ede8 ffff000001415218   ..$......RA.....
ffff80003a5b0120:  0000000000000040 0000000000010b88   @...............
ffff80003a5b0130:  0038004000000000 001d001e00400009   ....@.8...@.....
ffff80003a5b0140:  00010102464c457f 0000000000000000   .ELF............
ffff80003a5b0150:  0000000100b70003 00000000000010e0   ................
ffff80003a5b0160:  0000000000000040 0000000000026d78   @.......xm......
ffff80003a5b0170:  0038004000000000 001a001b00400007   ....@.8...@.....


```
于是对部分内容用sym进行了查看。

```bash
crash> sym ffff00000824edd8
ffff00000824edd8 (t) single_start /home/work_arm_lw2x08/flexbuild_lsdk1903_spl/packages/linux/linux/fs/seq_file.c: 541
crash> sym ffff00000824ee00
ffff00000824ee00 (t) single_stop /home/work_arm_lw2x08/flexbuild_lsdk1903_spl/packages/linux/linux/fs/seq_file.c: 552

```
明显是sequence文件的。

查看代码发现是 seq_ 函数，进一步推断这是个seq_opertaions结构。

![image](https://github.com/user-attachments/assets/d40ebc2f-ace5-43e0-bd33-a27e6515f622)



使用seq_operations结构查看内存。

```bash
crash> seq_operations ffff80003a5b0000 
struct seq_operations {
  start = 0xffff00000824edd8 <single_start>, 
  stop = 0xffff00000824ee00 <single_stop>, 
  next = 0xffff00000824ede8 <single_next>, 
  show = 0xffff000001415218 <TestMod_ProcContentSeqShow>  //业务模块函数
}

```

至此柳暗花明，那为什么会产生泄露呢？
查看相关业务代码发现，该文件的Open函数调用了single_open封装。但是释放的时候没有调用对应的single_release函数，而是直接调用了seq_release函数，导致single_open中申请的seq_operations内存没能释放，造成了内存泄漏。

```c
int single_release(struct inode *inode, struct file *file)
{
	const struct seq_operations *op = ((struct seq_file *)file->private_data)->op;
	int res = seq_release(inode, file);     //调用seq_release
	kfree(op);                              //释放seq_operations
	return res;
}
EXPORT_SYMBOL(single_release);
```

# 后记
本次内存泄漏难在查看slab entry中的内存内容。幸运的是很快找到了相同entry，且地址是代码地址，借此破译了泄露原因。
加入说slab entry中干扰性过多，或者内存内容特征不明显，那么分析起来要更加难上好几倍。

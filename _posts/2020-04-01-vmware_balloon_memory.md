---
title: "[Memory Leak] Hidden memory overuse caused by VMware's balloon memory"
date: 2020-04-01
---
公司测试人员告知一台vrrp测试设备内存占用过高，有内存泄漏嫌疑。我将《内存泄漏定位思路和方法》发给他，让他参照分析，正好也验证下文档是否好用，是否需要改进优化。

测试人员参照文档分析后，答复说应该是隐形内存占用过高。

我登上去后，大概过了一遍，排除了应用程序内存占用，slab内存占用，vmalloc内存占用等相关可能，确实是隐形内存占用过高，看来我的文档还是挺好用的 :-) 。

但是这次的隐形内存，不是socket队列积压导致，那么是什么原因导致呢？

这里犯了个错误，因为测试人员没有告诉我这是一台虚拟设备，我想当然认为是实体机，所以在定位上走了一些弯路。
尝试卸载公司业务模块，停止业务进程，发现无效果。
尝试停止了一些可疑的业务进程，无效果。
找不到新的怀疑方向，感到头大。
突然想到还有一种可能，发生在虚拟机上。宿主机通过balloon占用虚拟机内存。

关于虚拟化balloon技术，简单来说就是在虚拟机中存在一个气球(balloon)，气球中的内存是给宿主机使用的，虚拟机不能使用。

当宿主机需要内存时，可以请求虚拟机释放内存到气球中，也就是气球的充气膨胀。这样虚拟机的可用内存减少了，宿主机实际可用内存增大了。
反过来，虚拟机需要内存时，可以让气球放气缩小，也就是宿主机释放内存，让虚拟机有更多内存可使用。

balloon驱动就是直接alloc_pages操作内存，所以meminfo无法追踪到。

和测试人员确认设备是虚拟机后，基本可以断定是balloon充气，导致虚拟机系统内存被占用导致。
查看balloon驱动

```powershell
-bash-4.2# lsmod |grep balloon
vmw_balloon            18190  0 
```

执行卸载balloon驱动，验证是否ballon驱动导致。

```powershell
-bash-4.2# free -h	//查看内存占用，有大概1.2G隐形内存占用
              total        used        free      shared  buff/cache   available
Mem:           1.8G        1.2G         77M         31M        488M        369M
Swap:          1.6G         69M        1.5G
-bash-4.2# rmmod vmw_balloon	//卸载balloon
-bash-4.2# free -h		//查看内存占用，此时被占用的1.2G内存释放出来了
              total        used        free      shared  buff/cache   available
Mem:           1.8G         78M        1.2G         31M        488M        1.5G
Swap:          1.6G         69M        1.5G
```
经过验证，确认了是balloon导致。不是内存泄漏。

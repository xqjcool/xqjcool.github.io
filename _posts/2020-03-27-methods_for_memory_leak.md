---
title: "Ideas and Methods for Locating Memory Leaks"
date: 2020-03-27
---
本文主要针对基于Linux操作系统，提供了一种通用的内存泄漏定位分析思路和方法。

## 1. 查看内存概况

```powershell
[root@centos ~]# free
              total        used        free      shared  buff/cache   available
Mem:        1883844      376664       76136         192     1431044     1311252
Swap:       1048572       30540     1018032
```
通过free命令，我们对内存的整体使用有个初步了解，并快速是否存在内存泄漏可能。

### 1.1 used占用正常
used占用正常，free很低，但是buff/cache和available挺高。这种情况一般不是内存泄漏，而是可用内存被系统缓存起来了。
如需手动释放，可执行 echo x > /proc/sys/vm/drop_caches 将缓存释放出来。（x=1,2,3）

### 1.2 used占用过高
这种情况下存在内存泄漏可能，需要继续向下分析。


## 2. 检查/tmp目录
/tmp目录是占用内存空间的，有时候日志或者其他临时文件过大，导致内存占用过多。
尽早检查tmp目录可以帮我们快速排查这个问题，避免分析半天最后发现是tmp临时文件占用导致。

```powershell
[root@centos ~]# du -sh /tmp/*
0       /tmp/systemd-private-0bd2eb2d32be4029bb93736c12f9cfc6-httpd.service-93VBb6
0       /tmp/systemd-private-0bd2eb2d32be4029bb93736c12f9cfc6-ntpd.service-s017JD
0       /tmp/systemd-private-5245a0f476c64b80895d3fe3e5c5e9e6-httpd.service-Hp8PSy
0       /tmp/systemd-private-5245a0f476c64b80895d3fe3e5c5e9e6-ntpd.service-yQr1bn
0       /tmp/UNIX.domain
```

## 3. 检查进程的内存占用

```powershell
[root@centos ~]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
dbus       665  0.0  0.0  32660  1768 ?        Ssl  Mar24   2:29 /bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
root       672  0.0  0.0  24380  1856 ?        Ss   Mar24   1:01 /usr/lib/systemd/systemd-logind
root       886  0.0  0.0 110092   856 tty1     Ss+  Mar24   0:00 /sbin/agetty --noclear tty1 linux
root       893  0.0  0.0      0     0 ?        S<   Mar24   0:18 [kworker/1:1H]
root       918  0.0  0.0 126312  1716 ?        Ss   Mar24   2:28 /usr/sbin/crond -n
root      2722  0.0  0.1 146228  5232 ?        Rs   11:36   0:01 sshd: root@pts/2
root      2726  0.0  0.0 116008  2728 pts/2    Ss   11:36   0:00 -bash
root      4000  0.2  0.0      0     0 ?        S    13:26   1:10 [kworker/1:3]
root      5773  0.0  0.1 106012  4108 ?        Ss   15:48   0:00 /usr/sbin/sshd -D
root      5829  0.1  0.0 272000  2292 ?        Ssl  15:48   0:29 /usr/sbin/rsyslogd -n
root      5964  0.0  0.3 312716 11960 ?        Ss   15:48   0:02 /usr/sbin/httpd -DFOREGROUND
apache    5965  0.0  0.1 312716  6164 ?        S    15:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    5966  0.0  0.1 312716  6164 ?        S    15:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    5967  0.0  0.1 312716  6164 ?        S    15:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    5968  0.0  0.1 312716  6164 ?        S    15:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    5969  0.0  0.1 312716  6164 ?        S    15:48   0:00 /usr/sbin/httpd -DFOREGROUND
```
ps命令可以帮助我们定位分析应用进程内存泄漏。
观察RSS列，如果有进程RSS占用偏高，则存在内存泄漏可能。
需分析该程序源码是否存在泄漏可能。

如果应用进程没有泄漏，则继续向下分析。

## 4. meminfo查看内存信息

```powershell
[root@centos ~]# cat /proc/meminfo 
MemTotal:        1883844 kB
MemFree:          970208 kB
MemAvailable:    1321960 kB
Buffers:           51864 kB
Cached:           405336 kB
SwapCached:            0 kB
Active:           272120 kB
Inactive:         321892 kB
Active(anon):     137008 kB
Inactive(anon):      156 kB
Active(file):     135112 kB
Inactive(file):   321736 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       1048572 kB
SwapFree:        1048572 kB
Dirty:               288 kB
Writeback:             0 kB
AnonPages:        136824 kB
Mapped:            42852 kB
Shmem:               356 kB
Slab:              67860 kB
SReclaimable:      52156 kB
SUnreclaim:        15704 kB
KernelStack:        1824 kB
PageTables:         3736 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     1990492 kB
Committed_AS:     412576 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      223536 kB
VmallocChunk:   34359484404 kB
HardwareCorrupted:     0 kB
AnonHugePages:     83968 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       63480 kB
DirectMap2M:     2033664 kB
```

meminfo中对内存的使用进行了详细的统计，我们主要关注Slab和VmllocUsed。

### 4.1 检查Slab内存
#### 4.1.1 slub_nomerge已打开时的分析方法
Slab = SReclaimable + SUnreclaim
如果内存占用偏高，且SReclaimable偏高，则表示可用内存缓存在slab系统内，需要时可回收。
如果内存占用偏高，且SUnreclaim偏高，则表示可能存在内存泄漏。需要进一步观察slabinfo

```powershell
[root@centos ~]# cat /proc/slabinfo 
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
bridge_fdb_cache     128    128    128   32    1 : tunables    0    0    0 : slabdata      4      4      0
flow_cache             0      0    144   28    1 : tunables    0    0    0 : slabdata      0      0      0
ip_fib_trie          340    340     48   85    1 : tunables    0    0    0 : slabdata      4      4      0
ip_fib_alias         340    340     48   85    1 : tunables    0    0    0 : slabdata      4      4      0
ip_dst_cache         512    512    256   16    1 : tunables    0    0    0 : slabdata     32     32      0
PING                   0      0   1024   16    4 : tunables    0    0    0 : slabdata      0      0      0
RAW                   16     16   1024   16    4 : tunables    0    0    0 : slabdata      1      1      0
UDP                  120    120   1088   30    8 : tunables    0    0    0 : slabdata      4      4      0
tw_sock_TCP           64     64    256   16    1 : tunables    0    0    0 : slabdata      4      4      0
request_sock_TCP      64     64    256   16    1 : tunables    0    0    0 : slabdata      4      4      0
...//省略部分
```
基本上只要观察<active_objs> <num_objs> <objsize>前三个项即可，如果某个slab的<active_objs>异常的高，则很有可能存在泄漏。需要针对使用该slab的代码进行分析排查。

**注意：**
linux系统默认会将size接近的slab合并成一个slab，这样不利于分析slab内存泄漏。
为了方便定位，需要修改/boot/grub2/grub.cfg的系统启动参数，加上 slub_nomerge。

#### 4.1.2 slub_nomerge未打开时的分析方法 [2020.11.28增补]
slub_nomerge默认是未打开的，这时我们需要先查看是那些slab占用内存较高。然后再查看该slab和哪些slab共同使用，进而确认问题。
以郑州领未内存占用高问题举例：

##### 4.1.2.1 查看slab发现以下3个slab占用内存偏高，且申请objs数接近。

```bash
[root@cpe ~]# cat /proc/slabinfo
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
Acpi-ParseExt     1058636 1058680     72   56    1 : tunables    0    0    0 : slabdata  18905  18905      0
kmalloc-2048      1058194 1058288   2048   16    8 : tunables    0    0    0 : slabdata  66143  66143      0
kmalloc-256       1058462 1058544    256   16    1 : tunables    0    0    0 : slabdata  66159  66159      0
```

##### 4.1.2.2 查看和这些slab共用的slab。
```bash
[root@cpe ~]# cd /sys/kernel/slab/

//查看 Acpi-ParseExt真正使用的slab
[root@cpe slab]# ls -al | grep Acpi-ParseExt
lrwxrwxrwx   1 root root 0 Nov 27 15:20 Acpi-ParseExt -> :t-0000072

//查看 :t-0000072 被那些slab共用
[root@cpe slab]# ls -al|grep :t-0000072
lrwxrwxrwx   1 root root 0 Nov 27 15:20 Acpi-Operand -> :t-0000072
lrwxrwxrwx   1 root root 0 Nov 27 15:20 Acpi-ParseExt -> :t-0000072
lrwxrwxrwx   1 root root 0 Nov 27 15:20 eventpoll_pwq -> :t-0000072
lrwxrwxrwx   1 root root 0 Nov 27 15:20 TSPacket -> :t-0000072            //TSPacket也是占用72bytes
drwxr-xr-x   2 root root 0 Nov 27 15:20 :t-0000072


//查看 kmalloc-256 真正使用的slab 
[root@cpe slab]# ls -al | grep kmalloc-256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 dma-kmalloc-256 -> :dt-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 kmalloc-256 -> :t-0000256

//查看 :t-0000256 被那些slab共用
[root@cpe slab]# ls -al|grep :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 bio-0 -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 biovec-16 -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 filp -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 ip_dst_cache -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 kmalloc-256 -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 mnt_cache -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 nf_conntrack_expect -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 pool_workqueue -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 request_sock_TCP -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 request_sock_TCPv6 -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 sgpool-8 -> :t-0000256
lrwxrwxrwx   1 root root 0 Nov 27 15:20 skbuff_head_cache -> :t-0000256      //skbuff_head_cache使用
drwxr-xr-x   2 root root 0 Nov 27 15:20 :t-0000256
```

每个TSPacket对应1个skb， 并且这两个slab的objs数量接近， 基本确认是TSPacket未释放导致内存泄漏。
分析发现是业务模块存在一处错误逻辑导致TSPacket泄漏。


### 4.2 检查VmallocUsed内存
如果VmallocUsed内存占用偏高，则存在内存泄漏可能。需要进一步分析vmallocinfo。

```powershell
[root@centos ~]# cat /proc/vmallocinfo 
0xffffc90000000000-0xffffc90000201000 2101248 alloc_large_system_hash+0x171/0x239 pages=512 vmalloc N0=512
0xffffc90000201000-0xffffc90000302000 1052672 alloc_large_system_hash+0x171/0x239 pages=256 vmalloc N0=256
0xffffc90000302000-0xffffc90000304000    8192 acpi_os_map_memory+0xea/0x142 phys=fed00000 ioremap
0xffffc90000304000-0xffffc90000307000   12288 acpi_os_map_memory+0xea/0x142 phys=7fffe000 ioremap
0xffffc90000307000-0xffffc90000328000  135168 alloc_large_system_hash+0x171/0x239 pages=32 vmalloc N0=32
0xffffc90000328000-0xffffc90000369000  266240 alloc_large_system_hash+0x171/0x239 pages=64 vmalloc N0=64
0xffffc90000369000-0xffffc90000372000   36864 alloc_large_system_hash+0x171/0x239 pages=8 vmalloc N0=8
0xffffc90000372000-0xffffc9000037b000   36864 alloc_large_system_hash+0x171/0x239 pages=8 vmalloc N0=8
0xffffc9000037b000-0xffffc90000380000   20480 alloc_large_system_hash+0x171/0x239 pages=4 vmalloc N0=4
0xffffc90000380000-0xffffc90000382000    8192 pci_enable_msix+0x2f1/0x430 phys=febf1000 ioremap
0xffffc90000382000-0xffffc90000384000    8192 pci_enable_msix+0x2f1/0x430 phys=febf3000 ioremap
0xffffc90000385000-0xffffc900003a6000  135168 raw_init+0x41/0x141 pages=32 vmalloc N0=32
0xffffc900003a6000-0xffffc900003a8000    8192 swap_cgroup_swapon+0x48/0x160 pages=1 vmalloc N0=1
```
通过查看vmallocinfo，可以更详细的定位到是那个操作申请的内存过多，导致内存占用高，帮助我们定位问题。

## 5. 隐形内存占用
linux系统中有部分内存是通过alloc_pages/__get_free_page 直接申请的，/proc/meminfo无法统计到,这部分内存就像隐形了一样。

如果前面步骤还不能定位到问题，我们可以通过下面公式计算一下黑洞内存的大小：
MemFree + （Slab + VmallocUsed + PageTables + KernelStack + HardwareCorrupted + Bounce + Ghost） (Active + Inactive + Unevictable) + (HugePages_Total * Hugepagesize) = MemTotal
如果计算出来Ghost内存占用过大，基本上就是隐形内存占用了。

### 5.1 socket收发包队列积压
比较常见的就是数据包的申请会产生隐形内存占用。所以遇到隐形内存占用过高时，我们需要检查是否有socket的收包发包队列积压太多数据包。

```powershell
[root@centos ~]# netstat -noap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
...//省略部分
udp        0      0 0.0.0.0:67              0.0.0.0:*                           3240/dnsmasq         off (0.00/0/0)
udp        0      0 11.124.103.43:123       0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
udp        0      0 169.254.200.1:123       0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
udp        0      0 192.168.1.1:123         0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
udp        0      0 127.0.0.1:123           0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
udp        0      0 0.0.0.0:123             0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
udp6       0      0 fe80::84cf:68ff:fe8:123 :::*                                3130/ntpd            off (0.00/0/0)
udp6       0      0 ::1:123                 :::*                                3130/ntpd            off (0.00/0/0)
udp6       0      0 :::123                  :::*                                3130/ntpd            off (0.00/0/0)
raw6  76755840      0 :::58                   :::*                    7           3218/zebra           off (0.00/0/0)
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node   PID/Program name     Path
unix  2      [ ACC ]     STREAM     LISTENING     27149    3225/bgpd            /var/run/quagga/bgpd.vty
unix  2      [ ]         DGRAM                    9249     1/systemd            /run/systemd/notify
unix  2      [ ]         DGRAM                    25891    2871/rsyslogd        /run/systemd/journal/syslog
unix  2      [ ]         DGRAM                    9251     1/systemd            /run/systemd/cgroups-agent
unix  2      [ ACC ]     STREAM     LISTENING     25447    3229/ospfd           /var/run/quagga/ospfd.vty
...//省略部分
```
我们主要关注Recv-Q和Send-Q这两列，如果发现有积压太多数据包，则基本可以确定内存被积压的数据包给消耗掉了。
可以通过关闭相关服务或进程，看看内存是否恢复来确认。

### 5.2 pf_packet socket Recv-Q 积压数据包导致内存不足 [2020.11.28增补]
详见[pppd程序导致的内存泄漏](https://xqjcool.github.io/2020/11/26/memory_leak_pppd.html)
一些进程会启用packet socket收发二层协议，通过netstat是无法观察到的。使用ss协议可以观察到。

```bash
[root@cpe ~]# ss -pf link
Netid  Recv-Q Send-Q                                                                                          Local Address:Port                                                                                                           Peer Address:Port                
p_raw  10095232 0                                                                                                    ppp_disc:eth1                                                                                                                       *                      users:(("pppd",pid=17865,fd=10))
```
上面这个就是17865/pppd 进程收报队列积压了10095232 导致内存消耗过多。重启pppd服务后，占用内存可以释放掉。

### 5.3 vmware 虚拟机balloon导致 [2020.04.01增补]
关于虚拟化balloon技术，简单来说就是在虚拟机中存在一个气球(balloon)，气球中的内存是给宿主机使用的，虚拟机不能使用。

当宿主机需要内存时，可以请求虚拟机释放内存到气球中，也就是气球的充气膨胀。这样虚拟机的可用内存减少了，宿主机实际可用内存增大了。

反过来，虚拟机需要内存时，可以让气球放气缩小，也就是宿主机释放内存，让虚拟机有更多内存可使用。
balloon驱动就是直接alloc_pages操作内存，所以meminfo无法追踪到。

可以通过卸载balloon驱动来验证。

```powershell
-bash-4.2# free -h //查看内存占用，有大概1.2G隐形内存占用
              total        used        free      shared  buff/cache   available
Mem:           1.8G        1.2G         77M         31M        488M        369M
Swap:          1.6G         69M        1.5G
-bash-4.2# rmmod vmw_balloon //卸载balloon
-bash-4.2# free -h  //查看内存占用，此时被占用的1.2G内存释放出来了
              total        used        free      shared  buff/cache   available
Mem:           1.8G         78M        1.2G         31M        488M        1.5G
Swap:          1.6G         69M        1.5G
```
隐形内存占用释放，表示内存确实被balloon驱动占用了。


## 尾声
这是个快速定位内存问题的思路和方法。按照这个思路和步骤，基本上大多数内存问题都可以很快定位。希望能够帮助到有需要的朋友。

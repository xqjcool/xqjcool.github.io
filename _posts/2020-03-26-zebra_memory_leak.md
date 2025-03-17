---
title: "[Memory Leak] Excessive Hidden Memory Usage Caused by the Zebra Process"
date: 2020-03-26
---
这次问题源于线上设备发现内存使用过高，触发告警。技术支持报告后，登陆设备查看，发现系统内存占用（超过3.2G），导致可用内存只剩100多M。

```powershell
[root@ ~]# free
              total        used        free      shared  buff/cache   available
Mem:        3788284     3342156      110536      183316      335592       39432
Swap:       4063228       15696     4047532
```
正常运行时，系统也就占用内存600多M， 这一下子差出了2.6G左右的内存。

1. 首先查看应用进程，内存占用都在正常范围，没有发现内存占用过高进程。
排除了应用进程内存泄漏可能。

```powershell
[root@ ~]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 190692  3376 ?        Ss   Mar06   8:19 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    Mar06   0:00 [kthreadd]
root         3  0.6  0.0      0     0 ?        S    Mar06 190:52 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kworker/0:0H]
root         7  0.0  0.0      0     0 ?        S    Mar06   0:01 [migration/0]
root         8  0.0  0.0      0     0 ?        S    Mar06   0:00 [rcu_bh]
root         9  0.0  0.0      0     0 ?        S    Mar06   6:19 [rcu_sched]
root        10  0.0  0.0      0     0 ?        S    Mar06   0:01 [watchdog/0]
root        11  0.0  0.0      0     0 ?        S    Mar06   0:01 [watchdog/1]
root        12  0.0  0.0      0     0 ?        S    Mar06   0:01 [migration/1]
root        13  0.0  0.0      0     0 ?        S    Mar06   0:01 [ksoftirqd/1]
root        15  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kworker/1:0H]
root        16  0.0  0.0      0     0 ?        S    Mar06   0:01 [watchdog/2]
root        17  0.0  0.0      0     0 ?        S    Mar06   0:00 [migration/2]
root        18  0.0  0.0      0     0 ?        S    Mar06   0:26 [ksoftirqd/2]
root        20  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kworker/2:0H]
root        21  0.0  0.0      0     0 ?        S    Mar06   0:01 [watchdog/3]
root        22  0.0  0.0      0     0 ?        S    Mar06   0:01 [migration/3]
root        23  0.0  0.0      0     0 ?        S    Mar06   0:03 [ksoftirqd/3]
root        27  0.0  0.0      0     0 ?        S    Mar06   0:00 [kdevtmpfs]
root        28  0.0  0.0      0     0 ?        S<   Mar06   0:00 [netns]
root        29  0.0  0.0      0     0 ?        S    Mar06   0:00 [khungtaskd]
root        30  0.0  0.0      0     0 ?        S<   Mar06   0:00 [writeback]
root        31  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kintegrityd]
root        32  0.0  0.0      0     0 ?        S<   Mar06   0:00 [bioset]
root        33  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kblockd]
root        34  0.0  0.0      0     0 ?        S<   Mar06   0:00 [md]
root        41  0.0  0.0      0     0 ?        S    Mar06   0:16 [kswapd0]
root        42  0.0  0.0      0     0 ?        SN   Mar06   0:00 [ksmd]
root        43  0.0  0.0      0     0 ?        SN   Mar06   0:04 [khugepaged]
root        44  0.0  0.0      0     0 ?        S    Mar06   0:00 [fsnotify_mark]
root        45  0.0  0.0      0     0 ?        S<   Mar06   0:00 [crypto]
root        53  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kthrotld]
root        55  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kmpath_rdacd]
root        57  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kpsmoused]
root        59  0.0  0.0      0     0 ?        S<   Mar06   0:00 [ipv6_addrconf]
root        78  0.0  0.0      0     0 ?        S<   Mar06   0:00 [deferwq]
root       151  0.0  0.0      0     0 ?        S    Mar06   0:16 [kauditd]
root       286  0.0  0.0      0     0 ?        S<   Mar06   0:00 [ata_sff]
root       297  0.0  0.0      0     0 ?        S    Mar06   0:00 [scsi_eh_0]
root       298  0.0  0.0      0     0 ?        S<   Mar06   0:00 [scsi_tmf_0]
root       299  0.0  0.0      0     0 ?        S    Mar06   0:00 [scsi_eh_1]
root       300  0.0  0.0      0     0 ?        S<   Mar06   0:00 [scsi_tmf_1]
root       328  0.0  0.0      0     0 ?        S<   Mar06   0:05 [kworker/0:1H]
root       329  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kworker/2:1H]
root       389  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kdmflush]
root       390  0.0  0.0      0     0 ?        S<   Mar06   0:00 [bioset]
root       402  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kdmflush]
root       403  0.0  0.0      0     0 ?        S<   Mar06   0:00 [bioset]
root       416  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfsalloc]
root       417  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs_mru_cache]
root       418  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-buf/dm-0]
root       419  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-data/dm-0]
root       420  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-conv/dm-0]
root       421  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-cil/dm-0]
root       422  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-reclaim/dm-]
root       423  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-log/dm-0]
root       424  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-eofblocks/d]
root       425  0.0  0.0      0     0 ?        S    Mar06   5:33 [xfsaild/dm-0]
root       495  0.0  0.3  45012 14340 ?        Ss   Mar06   4:11 /usr/lib/systemd/systemd-journald
root       508  0.0  0.0 192624   836 ?        Ss   Mar06   0:00 /usr/sbin/lvmetad -f
root       524  0.0  0.0  43520  1120 ?        Ss   Mar06   0:00 /usr/lib/systemd/systemd-udevd
root       596  0.0  0.0      0     0 ?        S<   Mar06   0:01 [kworker/1:1H]
root       601  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-buf/sda1]
root       602  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-data/sda1]
root       603  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-conv/sda1]
root       604  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-cil/sda1]
root       605  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-reclaim/sda]
root       606  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-log/sda1]
root       607  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-eofblocks/s]
root       609  0.0  0.0      0     0 ?        S    Mar06   0:00 [xfsaild/sda1]
root       611  0.0  0.0      0     0 ?        S<   Mar06   0:00 [kdmflush]
root       612  0.0  0.0      0     0 ?        S<   Mar06   0:00 [bioset]
root       619  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-buf/dm-2]
root       620  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-data/dm-2]
root       621  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-conv/dm-2]
root       622  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-cil/dm-2]
root       623  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-reclaim/dm-]
root       624  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-log/dm-2]
root       625  0.0  0.0      0     0 ?        S<   Mar06   0:00 [xfs-eofblocks/d]
root       626  0.0  0.0      0     0 ?        S    Mar06   0:00 [xfsaild/dm-2]
root       637  0.0  0.0  55416  1376 ?        S<sl Mar06   0:42 /sbin/auditd -n
root       661  0.0  0.0  19300   856 ?        Ss   Mar06   3:13 /usr/sbin/irqbalance --foreground
dbus       671  0.0  0.0  24392  1040 ?        Ss   Mar06   6:52 /bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
root       691  0.0  0.0  24192  1384 ?        Ss   Mar06   3:30 /usr/lib/systemd/systemd-logind
root       697  0.0  0.0  31076   532 ?        Ss   Mar06   0:10 feeddog
root       886  0.0  0.0 110032   548 tty1     Ss+  Mar06   0:00 /sbin/agetty --noclear tty1 linux
root       887  0.0  0.0 110032   568 ttyS0    Ss+  Mar06   0:00 /sbin/agetty --keep-baud 115200 38400 9600 ttyS0 vt220
root       908  0.0  0.0 126248  1316 ?        Ss   Mar06   0:21 /usr/sbin/crond -n
root      2815  0.0  0.0 105956  3360 ?        Ss   Mar06   0:01 /usr/sbin/sshd -D
root      2871  0.0  0.0 271944  1496 ?        Ssl  Mar06   1:05 /usr/sbin/rsyslogd -n
root      3007  0.0  0.3 313328 11384 ?        Ss   Mar06   0:55 /usr/sbin/httpd -DFOREGROUND
ntp       3130  0.0  0.0  45116  2036 ?        Ss   Mar06   0:04 /usr/sbin/ntpd -u ntp:ntp -g
quagga    3218  0.0  0.0 113204  2292 ?        Ss   Mar06   0:00 /usr/sbin/zebra -d -A 127.0.0.1 -f /etc/quagga/zebra.conf
quagga    3225  0.0  0.0 115028  2792 ?        Ss   Mar06   0:06 /usr/sbin/bgpd -d -A 127.0.0.1 -f /etc/quagga/bgpd.conf
quagga    3229  0.0  0.0 115980  2800 ?        Ss   Mar06   0:00 /usr/sbin/ospfd -d -A 127.0.0.1 -f /etc/quagga/ospfd.conf
nobody    3240  0.0  0.0   6704   716 ?        Ss   Mar06   0:09 /usr/sbin/dnsmasq -k -K --conf-file=/etc/apx_dhcpd-mgmt.conf --port=0 -z
root      3279  0.0  0.0  80600  1344 ?        Ss   Mar06   0:03 pppd call eth1
nobody    4194  0.0  0.0   6704   716 ?        Ss   Mar13   0:12 /usr/sbin/dnsmasq -k -K --conf-file=/etc/apx_dhcpd.conf --port=0 -z
root      4515  0.0  0.0      0     0 ?        S    Mar24   0:29 [kworker/2:3]
root     10339  0.0  0.0      0     0 ?        S    Mar25   0:00 [kworker/u8:0]
root     16686  0.0  0.0      0     0 ?        S    05:40   0:05 [kworker/2:0]
root     19730  0.0  0.0      0     0 ?        S<   Mar24   0:00 [kworker/3:2H]
root     22743  0.0  0.0      0     0 ?        S    Mar15   0:14 [kworker/u8:2]
root     24565  0.0  0.0      0     0 ?        S    14:48   0:03 [kworker/1:2]
root     24763  0.0  0.0      0     0 ?        S<   Mar21   0:00 [kworker/3:0H]
root     24992  0.0  0.0      0     0 ?        S    15:17   0:02 [kworker/0:1]
root     26316  0.0  0.0      0     0 ?        S    16:50   0:00 [kworker/3:0]
root     26371  0.0  0.0      0     0 ?        S    16:54   0:00 [kworker/1:0]
root     26412  0.0  0.0      0     0 ?        S    16:57   0:00 [kworker/0:0]
root     26473  0.0  0.1 795820  4296 ?        Ssl  17:00   0:00 /bin/appex/frpc -c /etc/cpeagent/frpc.conf
root     26482  0.0  0.1 148860  5732 ?        Ss   17:00   0:00 sshd: root@pts/0
root     26484  0.0  0.0 115376  2040 pts/0    Ss+  17:00   0:00 -bash
root     26527  0.0  0.0      0     0 ?        S    17:01   0:00 [kworker/3:2]
root     26545  0.0  0.0      0     0 ?        S    17:02   0:00 [kworker/2:2]
root     26583  0.0  0.0      0     0 ?        S    17:05   0:00 [kworker/0:2]
root     26618  0.0  0.0      0     0 ?        S    17:07   0:00 [kworker/2:1]
root     26630  0.0  0.0      0     0 ?        S    17:08   0:00 [kworker/3:1]
root     26631  0.1  0.1 148860  5740 ?        Rs   17:09   0:00 sshd: root@pts/1
root     26644  0.0  0.0 115376  2008 pts/1    Ss   17:09   0:00 -bash
root     26662  0.0  0.0 151056  1824 pts/1    R+   17:09   0:00 ps aux
apache   32452  0.0  0.1 313328  6892 ?        S    Mar22   0:00 /usr/sbin/httpd -DFOREGROUND
apache   32453  0.0  0.1 313328  6892 ?        S    Mar22   0:00 /usr/sbin/httpd -DFOREGROUND
apache   32454  0.0  0.1 313328  6892 ?        S    Mar22   0:00 /usr/sbin/httpd -DFOREGROUND
apache   32455  0.0  0.1 313328  6892 ?        S    Mar22   0:00 /usr/sbin/httpd -DFOREGROUND
apache   32456  0.0  0.1 313328  6892 ?        S    Mar22   0:00 /usr/sbin/httpd -DFOREGROUND
```
2. 查看tmp目录
/tmp目录是占用内存空间的，有时候日志或者其他临时文件过大，导致内存占用过多。经查看/tmp目录基本没有占用空间。
排除tmp目录问题
```powershell
[root@ tmp]# du -sh /tmp/*
0       /tmp/systemd-private-cfc4ea0d3f364a30af065979ef4c25ce-httpd.service-NgFQL7
0       /tmp/systemd-private-cfc4ea0d3f364a30af065979ef4c25ce-ntpd.service-4WZ8gX
```

3. 查看meminfo，检查了Slab，VmallocUsed，等都算正常。
排除了我们内核模块内存泄漏可能。
但是丢失内存哪里去了呢？根据以下公式，我们可知
MemFree + （Slab + VmallocUsed + PageTables + KernelStack + HardwareCorrupted + Bounce + Ghost） (Active + Inactive + Unevictable) + (HugePages_Total * Hugepagesize) = MemTotal
内核使用alloc_pages/__get_free_page分配的内存，不会被meminfo统计到，像是幽灵一样，我们称之为Ghost内存，经过计算Ghost 约等于 2.6G。

```powershell
[root@ ~]# cat /proc/meminfo 
MemTotal:        3788284 kB
MemFree:          109316 kB
MemAvailable:      38224 kB
Buffers:               0 kB
Cached:           258864 kB
SwapCached:          408 kB
Active:           177444 kB
Inactive:         182300 kB
Active(anon):     137388 kB
Inactive(anon):   146808 kB
Active(file):      40056 kB
Inactive(file):    35492 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       4063228 kB
SwapFree:        4047532 kB
Dirty:                36 kB
Writeback:             0 kB
AnonPages:        100516 kB
Mapped:            27916 kB
Shmem:            183316 kB
Slab:              76904 kB
SReclaimable:      22104 kB
SUnreclaim:        54800 kB
KernelStack:        2464 kB
PageTables:         7052 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     5957368 kB
Committed_AS:     454600 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      641352 kB
VmallocChunk:   34358947836 kB
HardwareCorrupted:     0 kB
AnonHugePages:     61440 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       92580 kB
DirectMap2M:     4007936 kB
```
4. Ghost 内存占用的定位
这块一直没有好的办法，不过我分析内核 alloc_pages/__get_free_page 相关调用发现，数据包的申请会用到。
所以我查看数据包的占用情况。通过netstat命令，发现 3218/zebra 进程的Recv-Q竟然积压了76755840字节，这表明有大量数据包积压在socket的收包队列。而这些报文是网卡通过dma直接申请page占用的，所以在meminfo中追钟不到。基本确定是这个进程数据包积压过多导致。

```powershell
[root@ ~]# netstat -noap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
...//省略部分
udp        0      0 0.0.0.0:67              0.0.0.0:*                           3240/dnsmasq         off (0.00/0/0)
udp        0      0 61.170.203.43:123       0.0.0.0:*                           3130/ntpd            off (0.00/0/0)
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
执行 systemctl stop zebra，停止zebra服务。查看内存情况，已经恢复正常情况，used占用600M左右，free也恢复到了2.8G左右。

```powershell
[root@cpe tmp]# free
              total        used        free      shared  buff/cache   available
Mem:        3788284      619576     2859552      179964      309156     2787236
Swap:       4063228       18308     4044920
```
因为业务环境不需要zebra功能，所以暂时将其关闭。
不过zebra为什么会诡异的积压这么多数据包，还是个未解之谜，待有时间研究下。


尾声：
希望这篇文章对遇到内存问题，特别是遇到隐形内存占用而又苦于无法定位的朋友起到帮助作用。


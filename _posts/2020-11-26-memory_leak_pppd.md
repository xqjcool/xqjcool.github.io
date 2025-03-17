---
title: "[Memory Leak] Memory Leak Caused by pppd Program"
date: 2020-11-26
---
线上设备出现了内存泄漏，查看相关信息，发现是隐性内存占用过高导致。查阅[内存泄漏定位思路和方法](https://xqjcool.github.io/2020/03/27/methods_for_memory_leak.html)，没有找到问题原因。

查看线上内存历史统计，发现内存上涨很有规律，大概6天上涨到100%。然后因为内存不足，触发oom，杀掉工作进程后，内存可以下降下来。
![image](https://github.com/user-attachments/assets/692c5a37-9459-4db0-8399-341282217085)


设备断网的几天内存没有上涨。只有联网时内存消耗才上涨，推断问题和网络数据包相关。杀掉工作进程后内存可以下降，推断内存被缓存在某个位置，进程重启后释放。因为杀掉工作进城后可以释放。所以问题缩小到工作进程以及工作进程拉起的几个相关进程上。

挨个kill测试，发现kill pppd后，消耗的内存就会释放出来。确认是pppd进程导致。

pppd进程为何会消耗内存，且是meminfo无法跟踪到的隐性内存呢？ 看起来像zebra一样消息队列囤积导致，但是netstat却找不到对应队列。

经过调研，pppd会用到二层发现协议，会不会是二层协议处理socket囤包导致。于是使用ss命令查看。

```bash
[root@test ~]# ss -pf link
Netid  Recv-Q Send-Q                                                                                          Local Address:Port                                                                                                           Peer Address:Port                
p_raw  10095232 0                                                                                                    ppp_disc:eth1                                                                                                                       *                      users:(("pppd",pid=17865,fd=10))
```

发现进程pppd的packet socket Recv-Q 囤积了1千多万字节，因为网卡收报是直接申请页内存，并不会通过slab管理系统，所以meminfo无法观察到。并且网卡收包每个包都会占用比实际字节数更大的内存。

现在指导是packet socket积压数据包导致内存消耗过高。但是pppd为何没有处理这些数据包，而让他们囤积在socket队列中呢？

查阅ppp源码，发现是个bug，ppp的PAD会话socket，在完成PAD会话后，就不再使用。如果同一个子网内有其他设备发送PADI发送广播，则会被PAD会话socket收到，放到socket Recv-Q队列中，同时因为用户态socket不进行处理，那么队列就会越来越大，将内存消耗完。

ppp的github：https://github.com/paulusmack/ppp

于是做了个修改，规避了这个问题，并提交了个Pull request。
https://github.com/paulusmack/ppp/pull/179

----------------------------------2021/01/11补充---------------------------

pull request已经合入主线，大家可以直接使用pppd的最新代码即可。

```bash
commit 2b4166d02ed0e5dd74d37e2229805ccfd3dc23e0
Author: Xing Qingjie <88930741@qq.com>
Date:   Thu Nov 26 06:09:38 2020 -0500

    Close discovery socket after session completed
    
    After the session is complete, the socket is left unmanaged. When the
    interface receives PADIs from other device, the packets is putting
    in the socket's Recv-Q, which eat system memory.
    
    [root@test ~]# ss -f link
    Netid  Recv-Q Send-Q    Local Address:Port    Peer Address:Port
    p_raw  10269952 0          ppp_disc:eth1           *
    
    Signed-off-by: Xing Qingjie <88930741@qq.com>

```

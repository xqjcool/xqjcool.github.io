---
title: "Multicast fragmented packet loss issue after reassembly. PART-1"
date: 2022-09-29
---

# 问题现象

最近业务需要，将一个业务的传送从单播改为组播。修改后测试反馈收包丢包严重，直接影响业务功能。该报文是一个大包，分成多个分片发送到设备eth1口，重组后交给上层应用处理。

# 问题分析
## 查看网络错误统计
先是通过netstat查看报文统计，主要关注失败/丢包/错误相关统计。

```bash
root@test:~# netstat -s | grep -Ei "err|drop|fail"
    7 dropped because of missing route
    477139 fragments dropped after timeout. //计数持续上涨
    477139 packet reassemblies failed.      //计数持续上涨
    5606 input ICMP message failed
    0 ICMP messages failed
    5613 failed connection attempts
    683469 packet receive errors
    683469 receive buffer errors
    0 send buffer errors
    16 packets pruned from receive queue because of socket buffer overrun
root@test:~# netstat -s | grep -Ei "err|drop|fail"
    7 dropped because of missing route
    477168 fragments dropped after timeout.  //计数持续上涨
    477168 packet reassemblies failed        //计数持续上涨
    5606 input ICMP message failed
    0 ICMP messages failed
    5613 failed connection attempts
    683470 packet receive errors
    683470 receive buffer errors
    0 send buffer errors
    16 packets pruned from receive queue because of socket buffer overrun
```

首先发现报文重组失败计数持续上涨，进一步发现是重组超时失败导致的重组失败。
看了下重组超时时间设置是30s，这个时间足够收到所有分片。推断是部分分片报文没有在接口上收到，导致超时失败。

## 抓包分析
在eth1上使用tcpdump抓包，然后使用wireshark进行分析。
发现所有分片完整到达网络接口。协议中的counter值连续。确认接口没有丢包。
![image](https://github.com/user-attachments/assets/c1e8f791-0ca7-433c-8836-86c44a43c426)

## 业务代码问题？
为了排除业务代码的嫌疑，手动写了个组播报文接收测试程序，直接接收组播报文并打印报文序号。实际验证结果如下，可以发现序号不连续，存在严重丢包。可以确认非业务代码问题。

```bash
root@test:~# ./test 239.0.0.15 1067 10.84.0.42
UDP Server 0.0.0.0: 1067
recv from: 10.84.0.73:1067  898975  db79f
recv from: 10.84.0.73:1067  898976  db7a0
recv from: 10.84.0.73:1067  898978  db7a2
recv from: 10.84.0.73:1067  898980  db7a4
recv from: 10.84.0.73:1067  898982  db7a6
recv from: 10.84.0.73:1067  898984  db7a8
recv from: 10.84.0.73:1067  898986  db7aa
recv from: 10.84.0.73:1067  898988  db7ac
recv from: 10.84.0.73:1067  898989  db7ad
recv from: 10.84.0.73:1067  898990  db7ae
recv from: 10.84.0.73:1067  898991  db7af
recv from: 10.84.0.73:1067  898992  db7b0
recv from: 10.84.0.73:1067  898993  db7b1
recv from: 10.84.0.73:1067  898994  db7b2
```

## eth0和eth1在同一广播域
在分析中发现eth0上通过tcpdump也能收到radar组播报文。也就是说eth0和eth1在同一广播域。组播的每一个分片报文，eth0和eth1都会收到一份。

这是不合理的。因为eth0和eth1是不同网段，正常情况应该在不同的广播域。

在同一个广播域会导致分片报文在eth0和eth1都会收到，怀疑是这个原因导致radar丢帧。为了找到根因，进一步做了以下分析。

## ftrace日志分析
为了深入分析协议栈处理情况，我们使用ftrace对特定函数进行了追踪。
### 分片重组逻辑跟踪
可以看到分片报文在eth0和eth1上都有收到。以下图为例。eth1上收到第一个分片时创建重组队列qp，后续收到的分片（蓝框）放入队列中（arg1=0x0 表示成功）。eth0上收到（红框）的分片因为和eth1上收到（蓝框）的重叠冗余，导致这些报文被丢弃（arg1=0x1 表示失败）。

eth0继续收到分片（黄框），继续放入队列。因为所有分片都收集了，协议栈将其重组为一个完整的大包，并把qp释放。重组后的报文交由上层协议处理。

eth1收到了部分分片（蓝框），此时原有qp已经释放，协议栈将其当作一个新的分片，创建了一个新的qp，并将分片放入队列。

**因为该qp等不到其他报文，所以超时候会被丢弃。反映在网络统计中，就是重组超时失败计数持续增长。**
![image](https://github.com/user-attachments/assets/94af1df5-0c80-408e-8d96-ced05b3366b1)

在这里还差点被代码误导，看代码是根据dev进行查找，理论上eth0和eth1是不同的分片队列啊。
再一看红框里的代码实现，对于普通接口，返回值是0，也就是没有区分。:-)
![image](https://github.com/user-attachments/assets/584a49c3-5591-44f5-b3cf-b77c63a7322b)


### 重组后报文的丢弃逻辑跟踪
从上面分析我们可以看到虽然有重组失败计数增长，但仍然有一份重组后的报文送给了协议栈。以上图为例，这个报文重组后对应dev是eth0，而我们的组播是注册在eth1上的，eth0上没有注册。在后续处理中会将其丢弃。

为了验证我们的推断，使用ftrace跟踪协议栈 ip_rcv_finish_core 处理。我们发现确实如此。

以下图为例。可以看到所有从eth0送上的组播报文都被丢弃（ret=0x1）。另外对于序号d700a的报文，遇到了上面分析的冲突（即**同一报文的分片交叉到达eth0和eth1**），导致只有一份报文重组并且dev是eth0，被协议栈丢弃。导致应用没有收到。
![image](https://github.com/user-attachments/assets/48649aa8-bd74-4565-87f5-9162267f1861)

测试程序打印的日志，可以看到序号为d700a的报文没有收到。
![image](https://github.com/user-attachments/assets/ad93ce6e-db01-44ef-9faf-4c815d7f601e)


## 开启混杂模式？
如果不开启混杂模式的话，组播报文在二层就会被丢弃，不应该收到协议栈。
查看发现确实开了混杂模式，导致报文上到协议栈，引发了重组冲突。

```bash
root@test:~# cat /sys/class/net/eth0/flags 
0x1103    //bit8置位，说明在混杂模式
root@test:~# cat /sys/class/net/eth1/flags 
0x1103
```
![image](https://github.com/user-attachments/assets/347d9795-c18c-4580-b9d1-6eab57b5cd4b)


# 结论
## 问题原因
业务组播发送的分片报文，在eth0和eth1接口上都会收到（这是因为我们将eth0和eth1放在了同一个广播域中）。报文重组过程中会有冲突，eth1上部分报文重组失败，最终导致上层业务丢帧。
## 解决方法
关闭接口的混杂模式

# 相关技术栈链接
[Kprobe-based Event Tracing](https://docs.kernel.org/trace/kprobetrace.html)

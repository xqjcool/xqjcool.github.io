---
title: "Multicast fragmented packet loss issue after reassembly. PART-2"
date: 2022-10-04
---

# 组播分片报文重组后丢包问题（后续）
上篇组播分片报文重组后丢包问题，分析到最后是接口eth0和eth1在同一个广播域，且两个接口都处于混杂模式导致。

解决方案是关闭接口的混杂模式。

## 尝试清除IFF_PROMISC失败
于是想清除IFF_PROMISC标志，验证问题是否消失。

### 手动echo设置清除
手动清除失败，但其他标志位可以成功清除。看起来像是IFF_PROMISC被设定不可清除。

```bash
root@test:~# cat /sys/class/net/eth0/flags 
0x1103
root@test:~# echo 0x1003 > /sys/class/net/eth0/flags  //尝试清除IFF_PROMISC失败
root@test:~# cat /sys/class/net/eth0/flags 
0x1103
root@test:~# 
root@test:~# echo 0x1102 > /sys/class/net/eth0/flags   //清除IFF_UP成功
root@test:~# cat /sys/class/net/eth0/flags 
0x1102
root@test:~# echo 0x103 > /sys/class/net/eth0/flags   //清除IFF_MULTICAST成功
root@test:~# cat /sys/class/net/eth0/flags 
0x103
```

最为对比 eth2接口的IFF_PROMISC标志就可以成功设置和清除。

```bash
root@test:~# cat /sys/class/net/eth2/flags 
0x1003
root@test:~# echo 0x1103 > /sys/class/net/eth2/flags
root@test:~# cat /sys/class/net/eth2/flags 
0x1103
root@test:~# echo 0x1003 > /sys/class/net/eth2/flags
root@test:~# cat /sys/class/net/eth2/flags 
0x1003
```

### 通过ioctl接口设置清除
手动设置清除不行，于是尝试使用ioctl接口设置清除。写了个change工具通过ioctl设置SIOCGIFFLAGS。
经过验证，结果和手动设置相同。无法修改接口eth0和eth1的IFF_PROMISC标志，但可以修改其他标志位。

```bash
root@test:~# cat /sys/class/net/eth0/flags 
0x1103
root@test:~# ./change eth0 8
root@test:~# cat /sys/class/net/eth0/flags 
0x1103

root@test:~# ./change eth0 0
root@test:~# cat /sys/class/net/eth0/flags 
0x1102
root@test:~# ./change eth0 0
root@test:~# cat /sys/class/net/eth0/flags 
0x1103

root@test:~# ./change eth0 12
root@test:~# cat /sys/class/net/eth0/flags 
0x103
root@test:~# ./change eth0 12
root@test:~# cat /sys/class/net/eth0/flags 
0x1103
```

作为对比，eth0是可以设置清除IFF_PROMISC
```bash
//eth2 可以正常设置和清除IFF_PROMISC标志
root@test:~# cat /sys/class/net/eth2/flags 
0x1003
root@test:~# ./change eth2 8           //设置IFF_PROMISC
root@test:~# cat /sys/class/net/eth2/flags 
0x1103
root@test:~# ./change eth2 8          //清除IFF_PROMISC
root@test:~# cat /sys/class/net/eth2/flags 
0x1003
```

## 分析驱动代码
一度怀疑是驱动代码逻辑写死IFF_PROMISC不允许修改，随即就推翻了。因为eth2接口是相同的驱动，但IFF_PROMISC标志可以随意设置清除。
稳妥起见还是查阅了驱动代码，没有发现特殊逻辑。

## 虚拟vlan配置
固件团队分析到后面发现了原因。问题的引入来自于之前一次新增vlan操作。
eth0新增了eth0.10虚拟vlan，配置的 mac地址和eth0不一样，eth1新增了eth0.20虚拟vlan，配置的mac地址和eth1的mac不一样。
eth0 mac地址是70:b3:d5:20:00:21，但eth0.10地址是70:b3:d5:21:64:00。
eth1 mac地址是70:b3:d5:20:ff:21，但eth1.20地址是70:b3:d5:21:c8:01。

```bash
root@test:~# ifconfig eth0
eth0: flags=4419<UP,BROADCAST,RUNNING,MULTICAST>  mtu 8966
        inet 10.84.10.42  netmask 255.255.255.0  broadcast 10.42.10.255
        inet6 fe80::72b3:d5ff:fe20:21  prefixlen 64  scopeid 0x20<link>
        ether 70:b3:d5:20:00:21  txqueuelen 1000  (Ethernet)
        RX packets 224529948  bytes 178857830781 (178.8 GB)
        RX errors 0  dropped 3  overruns 0  frame 0
        TX packets 305526669  bytes 131688196359 (131.6 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@test:~# ifconfig eth0.100
eth0.10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 8966
        inet 192.168.10.42  netmask 255.255.255.0  broadcast 192.168.100.255
        inet6 fe80::72b3:d5ff:fe21:6400  prefixlen 64  scopeid 0x20<link>
        ether 70:b3:d5:21:64:00  txqueuelen 1000  (Ethernet)
        RX packets 16276987  bytes 6107515472 (6.1 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 275497881  bytes 117600578758 (117.6 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@test:~# ifconfig eth1
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1466
        inet 10.84.0.42  netmask 255.255.255.0  broadcast 10.42.0.255
        inet6 fe80::72b3:d5ff:fe20:ff21  prefixlen 64  scopeid 0x20<link>
        ether 70:b3:d5:20:ff:21  txqueuelen 1000  (Ethernet)
        RX packets 233679757  bytes 181307309357 (181.3 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 62101583  bytes 5165309302 (5.1 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@test:~# ifconfig eth1.200
eth1.20: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1466
        inet 192.168.20.42  netmask 255.255.255.0  broadcast 192.168.200.255
        inet6 fe80::72b3:d5ff:fe21:c801  prefixlen 64  scopeid 0x20<link>
        ether 70:b3:d5:21:c8:01  txqueuelen 1000  (Ethernet)
        RX packets 3808158  bytes 517024458 (517.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3828411  bytes 578738870 (578.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

由于eth0.10 的报文 要通过真实物理网口 eth0接收，而eth0的mac地址和eth0.10mac地址不同。为了能够接收发向eth0.10mac地址的报文。eth0需要设置成 IFF_PROMISC 模式。


检查多mac地址的代码逻辑在__dev_set_rx_mode中。
当dev->priv_flags没有设置 IFF_UNICAST_FLT时，会设置IFF_PROMISC标志。
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0a797fef8954e5ceb1b775b5c1df17c4.png)

至此，eth0和eth1为何IFF_PROMISC标志位设置的原因也找到了。

## 修改配置并验证

### 修改eth0.10 mac地址
手动修改eth0.10 mac地址，使其与eth0一致后。可以看到eth0的flags中 不再包含IFF_PROMISC标志。

```bash
root@test:~# cat /sys/class/net/eth0/flags 
0x1003
root@test:~# ifconfig eth0.10 down
root@test:~# ifconfig eth0.10 hw ether 70:b3:d5:20:00:21
root@test:~# ifconfig eth0.10 up
root@test:~# cat /sys/class/net/eth0/flags 
0x1003
```
### 验证组播
使用test程序验证，发现不再有丢帧问题。

```bash
root@test:~# ./test 239.0.0.15 1067 10.84.0.42
UDP Server 239.0.0.15: 1067
recv from: 10.84.0.73:1067  805583  c4acf
recv from: 10.84.0.73:1067  805584  c4ad0
recv from: 10.84.0.73:1067  805585  c4ad1
recv from: 10.84.0.73:1067  805586  c4ad2
recv from: 10.84.0.73:1067  805587  c4ad3
recv from: 10.84.0.73:1067  805588  c4ad4
recv from: 10.84.0.73:1067  805589  c4ad5
recv from: 10.84.0.73:1067  805590  c4ad6
recv from: 10.84.0.73:1067  805591  c4ad7
recv from: 10.84.0.73:1067  805592  c4ad8
recv from: 10.84.0.73:1067  805593  c4ad9
recv from: 10.84.0.73:1067  805594  c4ada
recv from: 10.84.0.73:1067  805595  c4adb
recv from: 10.84.0.73:1067  805596  c4adc
recv from: 10.84.0.73:1067  805597  c4add
```

验证通过。

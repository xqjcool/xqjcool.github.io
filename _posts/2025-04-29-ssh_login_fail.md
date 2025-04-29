---
title: "SSH login failure caused by an uninitialized structure variable."
date: 2025-04-29
---

# 结构变量未初始化导致的ssh登录失败

## 问题现象
QA上报说测试设备无法ssh连接，web页面是可以登录的。也就是说22端口连接异常，https的443端口没有问题。

## 初步观察
尝试ssh连接设备，被拒。

```bash
$ ssh admin@10.21.43.90 
ssh: connect to host 10.21.43.90 port 22: Connection refused
```

通过web页面管理进入设备shell，tcpdump观察报文交互。

```bash
/var/log# tcpdump -i eth0 tcp port 22
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:57:10.025407 IP 172.18.29.154.54090 > 10.21.43.90.22: Flags [S], seq 4105399158, win 64240, options [mss 1398,sackOK,TS val 456890810 ecr 0,nop,wscale 11], length 0
17:57:10.025444 IP 10.21.43.90.22 > 172.18.29.154.54090: Flags [R.], seq 0, ack 4105399159, win 0, length 0
```

可以看到tcp连接请求被rst了。那么是谁拒绝的sshd还是内核？因为什么原因呢？

对sshd进程进行trace，观察是否有accept处理。

```bash
/var/log# strace -p 4966
strace: Process 4966 attached
ppoll([{fd=3, events=POLLIN}, {fd=4, events=POLLIN}], 2, NULL, [], 8
```

ssh请求期间 进程处于poll等待，没有accept处理。也就是说报文并没有送给进程处理，那就是在内核中就被rst了。

## 继续深入

是iptables规则导致的么？查看没有发现异常规则。

```bash
/# iptables -A INPUT -s 172.18.29.154
/# iptables -nvL
Chain INPUT (policy ACCEPT 6918K packets, 561M bytes)
 pkts bytes target     prot opt in     out     source               destination    
25  1300            0    --  *      *       172.18.29.154       0.0.0.0/0
```

添加监视，发现报文进入了filter的INPUT链。

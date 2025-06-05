---
title: "Analysis of the significant performance degradation after upgrading to the new kernel"
date: 2025-06-05
---

# 升级新内核后性能大幅下降问题的分析

## 1. 问题起因

公司产品升级内核，从 linux 4.14 直接升级到 linux 6.1。 版本跨度太大，问题很多。好不容易稳定下来，测试发现性能下降太大。

http性能相比升级前下降70%左右，简直不敢相信。

## 2. 逐步分析

### 2.1 top和perf top

top观察发现CPU都跑满了，但是性能确实很差。
perf top 查看没有发现明显的瓶颈点。

![image](https://github.com/user-attachments/assets/d8f4f84e-7e98-46c4-b7ac-26c6a75edb20)

- 对比旧版本

请测试换上旧版本，进行对比测试。
发现同样是CPU跑满，但性能好了许多。

perf top 和 新版本的没有明显差异。

### 2.2 查看驱动信息

ethtool 查看i40e的驱动信息，并和旧版本对比。发现也没有明显差异。

### 2.3 查看tcp相关配置参数

```bash
find /proc/sys/ -name "*tcp*" | while read file; do
>   echo -n "$file: "
>   cat "$file"
> done
/proc/sys/net/ipv4/tcp_abort_on_overflow: 0
/proc/sys/net/ipv4/tcp_adv_win_scale: 1
/proc/sys/net/ipv4/tcp_allowed_congestion_control: reno bbr cubic
/proc/sys/net/ipv4/tcp_app_win: 31
/proc/sys/net/ipv4/tcp_autocorking: 1
/proc/sys/net/ipv4/tcp_available_congestion_control: reno bbr cubic
//省略部分
```

和旧版对比，也基本一致。

### 2.4 查看CONFIG相关

怀疑新版本开启了什么DEBUG功能导致性能下降。

于是对比新旧版本内核 config配置，发现差异不大。没有明显的消耗性能的DEBUG配置。

### 2.5 网卡中断

查看网卡中断信息，观察到中断是均匀分布在各个核上。

## 3. 柳暗花明

从上面分析看来，每个地方都没有问题，但性能却下降了那么多。
而且注意到，虽然性能下降了，但是top中 user/sys/nic/idle/io/irq/sirq的比例却是一致的。
好像新版是一个低端的旧版设备。

偶然观察cpu信息，发现主频出奇的低。

```bash
# cat /proc/cpuinfo | more
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 85
model name      : Intel(R) Xeon(R) Silver 4215R CPU @ 3.20GHz  //标配 3.2GHz
stepping        : 7
microcode       : 0x5000024
cpu MHz         : 999.848      //实际运行 999.848MHz 不到1GHz
cache size      : 11264 KB

```

这么低的主频，性能自然上不去了。

于是查看CPU的调速策略。

```bash
/# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
userspace
```

怪不得cpu主频这么低。

```bash
/# echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
performance

/# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance

/# cat /proc/cpuinfo | more
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 85
model name      : Intel(R) Xeon(R) Silver 4215R CPU @ 3.20GHz
stepping        : 7
microcode       : 0x5000024
cpu MHz         : 3199.996
```

手动调整到performance后，发现系统性能瞬间跃升，和旧版一致。

## 4. 寻根问底

经对比，旧版的默认 scaling_governor 是 performance， 而新版的默认 scaling_governor 是 userspace。

查看内核config，发现旧版 

```bash
CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y
```

而新版

```bash
CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y
```

问题出在这个CONFIG上，但是编译时的初始config并没有做什么改变啊。

查看初始config发现，里面同时配置了

```bash
CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y  //原始config
CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y  //merge用的特定型号的config
```

旧版内核merge后，保留了CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y,所以功能正常。
新版内核merge后，保留了CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y，所以性能下降。


将干扰项 CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y 删除后，编译版本加载后，性能正常。

## 5. 后记

升级版本性能下降70%左右，也是够吓人的。不过越是性能下降明显，越说明不是代码导致。更有可能是配置或者选项错误导致。

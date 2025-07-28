---
title: "The system fails to generate a coredump file after crashing"
date: 2025-07-28
---

# 系统crash后无法产生coredump文件

## 1. 问题现象

最近遇到几起产品系统crash后无法产生coredump的问题。但问题现象不尽相同。

## 2. crashkernel size太小导致

crashkernel 的作用是在系统启动时预留一块物理内存区域，供第二内核（crash kernel）在主内核崩溃时使用，以便收集 coredump（vmcore）用于诊断分析。
crashkernel size太小会导致initramfs无法启动等问题，最终导致无法生成coredump文件。

公司B产品就是因为crashkernel size太小，系统内存16G，但是crashkernel=192M。改为512M后，能够正常产生coredump文件。

大体上crashkernel和系统内存的对照(参考 RHEL/CentOS/Oracle Linux 等)

| 系统总内存（RAM）  | 建议 `crashkernel` 大小 |
| ----------- | ------------------- |
| ≤ 2 GB      | `128M`              |
| 2 – 4 GB    | `256M`              |
| 4 – 64 GB   | `512M` – `768M`     |
| 64 – 128 GB | `1G`                |
| > 128 GB    | `2G` 或更高          |

## 3. makedumpfile版本太旧

这次升级内核后遇到了无法产生coredump文件的问题。起初以为是crashkernel size问题，但调大size后，仍然不能正常工作。
随后注意到串口有如下打印：

```bash
The kernel version is not supported.
The makedumpfile operation may be incomplete.
check_release: Can't get the kernel version.

makedumpfile Failed.
```

从打印内容推断makedumpfile和新内核不兼容，导致makedumpfile无法正常工作。
于是下载了新版的makedumpfile，并交叉编译，替换原因makedumpfile。

编译image，并加载。测试发现coredump可以正常产生了。

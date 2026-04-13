---
title: "ebpf课程1：初识 ebpf"
date: 2026-04-13
---

# EBPF课程1：初识 EBPF

## 1. 背景知识

### 1.1 什么是eBPF

eBPF（extended Berkeley Packet Filter）是一种运行在 Linux 内核中的安全、可编程虚拟机机制，允许用户在不修改内核的情况下动态加载程序，用于网络、观测和安全等场景。

### 1.2 核心本质

- 是一种“内核里的程序”
- 是一个“受限制的虚拟机”
- 是“可插拔的 hook 机制”

### 1.3 eBPF能做什么

网络处理

- DDoS 防护
- NAT/LB
- 流量统计
- 打 mark/policy routing

可观测性

- 性能分析
- tracing
- 延迟分析

安全

- LSM hook
- 系统调用控制
- 容器安全

## 2. 简单的eBPF程序示例

### 2.1 eBPF code

一个简单的tracepoint eBPF 代码

```
  //tp_execve.bpf.c
  // SPDX-License-Identifier: GPL-2.0
  #include <linux/bpf.h>                                //BPF 基础类型定义
  #include <bpf/bpf_helpers.h>                          //SEC() 和 helper 声明

  struct trace_event_raw_sys_enter {
      __u64 id;
      __u64 args[6];
  };

  SEC("tracepoint/syscalls/sys_enter_execve")
  int tp_execve(struct trace_event_raw_sys_enter *ctx)
  {
      __u64 pid_tgid = bpf_get_current_pid_tgid();
      __u32 pid = pid_tgid >> 32;

      char fmt[] = "execve pid=%u\n";
      bpf_trace_printk(fmt, sizeof(fmt), pid);
      return 0;
  }

  char LICENSE[] SEC("license") = "GPL";

```

编译命令

```
clang -O2 -g -target bpf -c tp_execve.bpf.c -o tp_execve.bpf.o \
    -I/kernel_path/tools/lib/bpf \
    -I/kernel_header_path/include

```

用户态加载程序

```
  // SPDX-License-Identifier: GPL-2.0
  #include <stdio.h>
  #include <stdlib.h>
  #include <bpf/libbpf.h>
  #include <bpf/bpf.h>

  int main(void)
  {
      struct bpf_object *obj = NULL;
      struct bpf_program *prog = NULL;
      struct bpf_link *link = NULL;
      int err;

      obj = bpf_object__open_file("tp_execve.bpf.o", NULL);    //打开 BPF ELF 文件（还没加载进内核），解析其中的程序和 map 定义
      if (!obj) {
          fprintf(stderr, "open failed\n");
          return 1;
      }

      err = bpf_object__load(obj);                              //加载到内核，加载到内核、校验并加载 BPF 程序、失败会返回 verifier 错误
      if (err) {
          fprintf(stderr, "load failed: %d\n", err);
          return 1;
      }

      prog = bpf_object__find_program_by_name(obj, "tp_execve");   //从 ELF 里找出名字为 tp_execve 的 BPF 程序
      if (!prog) {
          fprintf(stderr, "find prog failed\n");
          return 1;
      }

      link = bpf_program__attach_tracepoint(prog, "syscalls", "sys_enter_execve");  //把该 BPF 程序 挂载到 tracepoint。返回 bpf_link 用于后续 detach
      if (!link) {
          fprintf(stderr, "attach failed\n");
          return 1;
      }

      printf("attached, press Enter to exit...\n");
      getchar();

      bpf_link__destroy(link);                                    //解除挂载（detach）
      bpf_object__close(obj);                                      //释放用户态资源
      return 0;
  }

```

编译方式

```
gcc -O2 -g tp_execve_user.c -o tp_execve_user \
    -lbpf -lelf -lz
```

## 3 后记

通过 给出 tracepoint BPF 程序 配合 用户态 loader 代码，对eBPF的代码和加载操作有了初步的了解。

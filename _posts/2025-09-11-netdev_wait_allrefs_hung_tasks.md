---
title: "Analysis of the softlockup issue caused by the net_device refcnt not being 1"
date: 2025-09-11
---

# 网络设备的索引不为1，导致的softlockup的问题分析

## 1. 问题现象

测试发现设备在升级过程中，耗时较长，并且升级失败。检查发现系统出现crash。
原因是 "Kernel panic - not syncing: softlockup: netdev_wait_allrefs hung tasks"。
设备在unreigster过程中 因为 索引不为1，导致迟迟不能释放，最终触发softlockup告警，最终触发内核panic(代码中增加了超时3分钟触发panic处理)。 


对于这类型问题，需要先从日志中获取 问题net_device的相关信息。

```bash
[ 1056.498243] unregister_netdevice: waiting for vlan1 to become free. Usage count = 7

```

从这条日志可以得出， net_device vlan1 当前有7个refcnt， 有6个异常refcnt没有释放(正常在这里refcnt应该为1) 。

然后取coredump文件，使用crash工具调试。

```bash
      KERNEL: vmlinux  [TAINTED]
    DUMPFILE: coredump-2025-09-04-10_59  [PARTIAL DUMP]
        CPUS: 2
        DATE: Thu Sep  4 10:59:21 PDT 2025
      UPTIME: 00:19:04
LOAD AVERAGE: 0.98, 0.89, 0.63
       TASKS: 78
    NODENAME: TESTMODEL
     RELEASE: 6.1
     VERSION: #1 SMP Tue Sep  2 16:39:22 PDT 2025
     MACHINE: x86_64  (2599 Mhz)
      MEMORY: 16 GB
       PANIC: "Kernel panic - not syncing: softlockup: netdev_wait_allrefs hung tasks"
crash> bt
PID: 4962     TASK: ffff8881025dc880  CPU: 0    COMMAND: "mdsvr"
 #0 [ffffc900022e7ae8] machine_kexec at ffffffff802857d3
 #1 [ffffc900022e7b68] __crash_kexec at ffffffff80394297
 #2 [ffffc900022e7c38] panic at ffffffff802a4da5
 #3 [ffffc900022e7cb8] netdev_run_todo at ffffffff80cb82bc
 #4 [ffffc900022e7d58] vlan_ioctl_handler at ffffffff810590d3
 #5 [ffffc900022e7df8] sock_ioctl at ffffffff80c6c8be
 #6 [ffffc900022e7e68] __x64_sys_ioctl at ffffffff8052e795
 #7 [ffffc900022e7f18] do_syscall_64 at ffffffff810942a9
 #8 [ffffc900022e7f50] entry_SYSCALL_64_after_hwframe at ffffffff812000dc
    RIP: 00007f95312478d8  RSP: 00007ffd66effb28  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 0000000000000010  RCX: 00007f95312478d8
    RDX: 00007ffd66effb60  RSI: 0000000000008983  RDI: 0000000000000010
    RBP: 0000000000000001   R8: 0000000000000004   R9: 0000000000000007
    R10: 00007ffd66effb1c  R11: 0000000000000246  R12: 00007ffd66effb60
    R13: 0000000000000000  R14: 00007ffd66f00260  R15: 0000000000000000
    ORIG_RAX: 0000000000000010  CS: 0033  SS: 002b

```

因为这种crash 没有保留当时的寄存器，给问题定位带来了一些困难。
这时通过 net命令已经看不到 vlan1 设备了，所以我们需要通过栈信息来获取到 vlan1的地址。

```bash
crash> rd -s ffffc900022e7c38 128
ffffc900022e7c38:  panic+326        0000003000000008 
ffffc900022e7c48:  ffffc900022e7cc0 ffffc900022e7c60 
ffffc900022e7c58:  _printk+89       schedule_timeout+210 
ffffc900022e7c68:  00000000ffffdfff ffffc900022e7bc0 
ffffc900022e7c78:  d66ccc72ee3a4e00 0000000000001fff 
ffffc900022e7c88:  _printk_rb_static_descs ffffc900022e7d10
ffffc900022e7c98:  ffffc900022e7d00 ffffc900022e7d00
ffffc900022e7ca8:  00000001000cdf18 ffffc900022e7d50
ffffc900022e7cb8:  netdev_run_todo+2220 0000000000000002   //    rsp-0x68
ffffc900022e7cc8:  0000000000000002 vlan_vid_del+285 
ffffc900022e7cd8:  0000000000000002 00000001000ce1f1 
ffffc900022e7ce8:  ffff8881025dc880 000000fa03a45440 
ffffc900022e7cf8:  00000001000cc919 ffff888102af84d0 
ffffc900022e7d08:  ffff888102af84d0 ffff888102af8000       //  net_device
ffffc900022e7d18:  0000000000000000 d66ccc72ee3a4e00 
ffffc900022e7d28:  ffff888102af8000 00007ffd66effb60        //rbx  r12 
ffffc900022e7d38:  ffff888103a45440 0000000000000000        //r13  r14
ffffc900022e7d48:  init_net         ffffc900022e7df0        //r15  rbp
ffffc900022e7d58:  vlan_ioctl_handler+1411 ffff8881025dc880 

```

从栈中得到 vlan1的net_device地址 ffff888102af8000

```bash
crash> net_device.name,pcpu_refcnt -x ffff888102af8000
  name = "vlan1\000\000\000\000\000",
  pcpu_refcnt = 0x607bd000b234,

crash> atomic_t 0x607bd000b234:a
[0]: ffffe8ffffc0b234
struct atomic_t {
  counter = -12
}
[1]: ffffe8ffffd0b234
struct atomic_t {
  counter = 19
}
```

refcnt=19-12 =7
内存信息和日志信息相互印证。

## 2. 初步分析

定位这种设备索引不为1的问题，相比其他crash要困难。因为此时你只能看到索引不为1，但是谁hold住了这个设备的索引，是没有地方可以查看的。

直接dev_hold和dev_put日志的话，会因为日志太多而无法分析。于是将日志限制在针对 vlan1 这个设备的 hold和put操作进行栈信息打印。
编译debug image，加载复现后。收集日志，进行逐一比对(有时候就是需要这种纯手工劳动)。

```bash
//部分省略
=====
[12927.660926] [__dev_hold:4073] dev:vlan1 refcnt: 27 		//dst_init * 15次 +

[12927.668936] [__dev_put:4057] dev:vlan1 refcnt: 29 		//dst_destroy	* 7次 -
[13293.869949] [__dev_put:4057] dev:vlan1 refcnt: 29 		//dst_dev_put * 8次 -
===
[12927.660965] neighbour: [__dev_hold:4073] dev:vlan1 refcnt: 28 		//___neigh_create * 4次 +

[13293.862636] neighbour: [__dev_put:4057] dev:vlan1 refcnt: 33 		//neigh_destroy		* 4次 -
====
[12927.625415] [__dev_hold:4073] dev:vlan1 refcnt: 12 	//fib_create_info+0x96f	* 1次 +
[12927.625483] [__dev_hold:4073] dev:vlan1 refcnt: 13 	//fib_check_nh+0xa31 * 3次 +
[12927.627085] [__dev_hold:4073] dev:vlan1 refcnt: 16 	//fib_check_nh+0x9c3	* 11次 +
[12927.627562] [__dev_hold:4073] dev:vlan1 refcnt: 18 	//fib_nh_dup+0x39d		* 8次  +

[12927.632924] [__dev_put:4057] dev:vlan1 refcnt: 24 		//fib_nh_common_release+0x29c * 17次 -
=======
```

经过筛选和汇总，发现 fib_info相关对 vlan1 引用了 23次，但是释放了17次，有6个引用没有释放！！！

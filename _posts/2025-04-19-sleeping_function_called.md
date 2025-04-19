---
title: "BUG: sleeping function called from invalid context"
date: 2025-04-19
---

# 在无效上下文中调用了可能导致休眠的函数

最近在做产品内核升级，遇到个"scheduling while atomic"问题，定位还颇费了一些功夫，特记录下来，希望能给大家带来一些帮助。

## 问题起因
先是自动化测试case失败，提示有进程segfault，观察系统日志，发现还有"BUG: scheduling while atomic"。进程brctl的segfault大概率是这个BUG引起的。

```bash
Apr 15 11:02:05 VM kern.info kernel: [  320.682331] brctl[3988]: segfault at 7f2c1fe4dda8 ip 00007f2c1fe4dda8 sp 00007ffc6e954548 error 14 in libcrypt.so.1[7f2c1fe22000+2c000] likely on CPU 0 (core 0, socket 0)
Apr 15 11:02:05 VM kern.info kernel: [  320.682368] Code: Unable to access opcode bytes at 0x7f2c1fe4dd7e.

Apr 15 11:02:05 VM kern.err kernel: [  320.682391] BUG: scheduling while atomic: brctl/3988/0x00000002
Apr 15 11:02:05 VM kern.warn kernel: [  320.682451] no locks held by brctl/3988.
Apr 15 11:02:05 VM kern.warn kernel: [  320.682453] Modules linked in: nf_conntrack(O)
Apr 15 11:02:05 VM kern.warn kernel: [  320.682466] CPU: 0 PID: 3988 Comm: brctl Kdump: loaded Tainted: G        W  O       6.1 #33
Apr 15 11:02:05 VM kern.warn kernel: [  320.682471] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
Apr 15 11:02:05 VM kern.warn kernel: [  320.682472] Call Trace:
Apr 15 11:02:05 VM kern.warn kernel: [  320.682474]  <TASK>
Apr 15 11:02:05 VM kern.warn kernel: [  320.682475]  dump_stack_lvl+0x76/0xaa
Apr 15 11:02:05 VM kern.warn kernel: [  320.682480]  dump_stack+0x10/0x16
Apr 15 11:02:05 VM kern.warn kernel: [  320.682483]  __schedule_bug.cold+0x5f/0x75
Apr 15 11:02:05 VM kern.warn kernel: [  320.682486]  __schedule+0x9c2/0xbd0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682489]  schedule+0x5f/0xe0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682493]  schedule_timeout+0x163/0x1b0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682496]  __wait_for_common+0xb7/0x1e0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682499]  ? usleep_range_state+0xc0/0xc0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682502]  wait_for_completion_state+0x21/0x40
Apr 15 11:02:05 VM kern.warn kernel: [  320.682505]  call_usermodehelper_exec+0x178/0x1b0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682509]  do_coredump+0x930/0x16a0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682515]  get_signal+0xa2e/0xd50
Apr 15 11:02:05 VM kern.warn kernel: [  320.682519]  ? force_sig_info_to_task+0xc2/0x180
Apr 15 11:02:05 VM kern.warn kernel: [  320.682523]  arch_do_signal_or_restart+0x45/0x710
Apr 15 11:02:05 VM kern.warn kernel: [  320.682529]  exit_to_user_mode_prepare+0x15d/0x250
Apr 15 11:02:05 VM kern.warn kernel: [  320.682532]  irqentry_exit_to_user_mode+0x9/0x30
Apr 15 11:02:05 VM kern.warn kernel: [  320.682535]  irqentry_exit+0x80/0xb0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682538]  exc_page_fault+0x2da/0x720
```

初步推测是某个函数做了preempt_count add，忘了做 dec了。但是从调用栈上看不出来。

查看系统日志发现在此之前还有个"BUG: sleeping function called from invalid context"

```bash
Apr 15 11:02:05 VM kern.err kernel: [  320.681589] BUG: sleeping function called from invalid context at kernel/rseq.c:132
Apr 15 11:02:05 VM kern.err kernel: [  320.681830] in_atomic(): 1, irqs_disabled(): 0, non_block: 0, pid: 3988, name: brctl
Apr 15 11:02:05 VM kern.err kernel: [  320.681940] preempt_count: 1, expected: 0
Apr 15 11:02:05 VM kern.warn kernel: [  320.681984] no locks held by brctl/3988.    //说明没有持有任何锁，也就不是lock/unlock不匹配导致的问题
Apr 15 11:02:05 VM kern.warn kernel: [  320.681987] CPU: 0 PID: 3988 Comm: brctl Kdump: loaded Tainted: G           O       6.1 #33
Apr 15 11:02:05 VM kern.warn kernel: [  320.681990] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 12/12/2018
Apr 15 11:02:05 VM kern.warn kernel: [  320.681992] Call Trace:
Apr 15 11:02:05 VM kern.warn kernel: [  320.681994]  <TASK>
Apr 15 11:02:05 VM kern.warn kernel: [  320.681996]  dump_stack_lvl+0x76/0xaa
Apr 15 11:02:05 VM kern.warn kernel: [  320.682005]  dump_stack+0x10/0x16
Apr 15 11:02:05 VM kern.warn kernel: [  320.682008]  __might_resched.cold+0xd2/0xe3
Apr 15 11:02:05 VM kern.warn kernel: [  320.682012]  __might_sleep+0x42/0x70
Apr 15 11:02:05 VM kern.warn kernel: [  320.682018]  __might_fault+0x31/0x90
Apr 15 11:02:05 VM kern.warn kernel: [  320.682022]  __rseq_handle_notify_resume+0x8a/0x550
Apr 15 11:02:05 VM kern.warn kernel: [  320.682027]  ? exit_to_user_mode_prepare+0x11d/0x250
Apr 15 11:02:05 VM kern.warn kernel: [  320.682032]  exit_to_user_mode_prepare+0x198/0x250
Apr 15 11:02:05 VM kern.warn kernel: [  320.682034]  ? vfs_fileattr_set+0x230/0x230
Apr 15 11:02:05 VM kern.warn kernel: [  320.682039]  syscall_exit_to_user_mode+0x2d/0x60
Apr 15 11:02:05 VM kern.warn kernel: [  320.682043]  do_syscall_64+0x99/0xe0
Apr 15 11:02:05 VM kern.warn kernel: [  320.682045]  entry_SYSCALL_64_after_hwframe+0x64/0xce
Apr 15 11:02:05 VM kern.warn kernel: [  320.682050] RIP: 0033:0x7f2c1fd04a38
```

`in_atomic(): 1, irqs_disabled(): 0, non_block: 0, pid: 3988, name: brctl` 表示此时的preempt_count为1，中断没有禁止，进程是 brctl。
`preempt_count: 1, expected: 0` preempt_count为1， 期望为0
`no locks held by brctl/3988.`表示不是lock/unlock不匹配导致的问题。
综上猜测是某个地方调用preempt_disable后没有enableb导致的。

## 添加debug log追踪

因为从栈信息看不出是哪里出的问题，所以在相关路径添加了preempt_count记录，在出问题时将其打印，用来找出在哪里异常的。
这里开始时走了些弯路，preempt_count记录用的普通全局变量，验证发现日志打印不太合理。
后来想起来preempt_count是per cpu的，将跟踪记录也改成per cpu变量后，日志正确了。确认了是系统调用前prept_count还是0，调用后preempt_count变成1了。

```bash
[do_syscall_x64:53] p1:0x0 p2:0x0 p3:0x0 p4:0x0 p5:0x0 0x1 unr:16 s:vfs_fileattr_set+0x230/0x230 
```

本想直接打印系统调用函数的，没想到打印出来`vfs_fileattr_set`并不是系统调用。 不过没关系，通过unr，我们查到这是 ioctl 系统调用。

brctl的ioctl有很多cmd，是哪个导致的呢？

## debug trace

其实在br_ioctl_stub中添加debug log，编译image 调试也能得到，但是相对耗时多。于是决定使用debug trace来快速得到。

```bash
echo 0 > events/syscalls/enable
echo 1 > events/syscalls/sys_enter_ioctl/enable
echo 1 > events/syscalls/sys_exit_ioctl/enable

echo 1 > tracing_on
```

跑完case后， 查看trace 日志，有超多ioctl的系统调用，但是过滤 brctl进程后，只有一个。

```bash
/sys/kernel/debug/tracing# cat trace |  grep brctl
           brctl-5524    [000] ...1.  4307.349482: sys_ioctl(fd: b, cmd: 89a1, arg: 7ffc0003efaf)
           brctl-5524    [000] ...1.  4307.349483: __x64_sys_ioctl <-do_syscall_64
           brctl-5524    [000] ...2.  4307.360536: sys_ioctl -> 0x0
```


89a1 对应
```c
#define SIOCBRDELBR     0x89a1		/* remove bridge device         */
```

也就是 `brctl delbr br0` 命令引发的系统调用。

手动验证,确实如此。

```bash
/sys/kernel/debug/tracing# brctl addbr br0
/sys/kernel/debug/tracing# ip link set br0 up
/sys/kernel/debug/tracing# ip link set br0 down
/sys/kernel/debug/tracing# brctl delbr br0
Segmentation fault (core dumped)
```

## 水落石出

在 `br_del_bridge` 中加调试日志观察，缩小范围后，配合代码找到了下面这块patch代码。

```c
#ifdef DEVICE_REF_DEBUG
    if (unlikely(get_cpu_var(dev_ref_debug_enable))) {
        if (dev->ref_debug) {
            kfree(dev->ref_debug);
            dev->ref_debug = NULL;
        }
    }
#endif
```

终于真相大白。 问题出在 `get_cpu_var(dev_ref_debug_enable)` 这里。这个宏会调用 `preempt_disable()`，将 preempt_count 加1 。最终导致BUG发生。

```c
/*
 * Must be an lvalue. Since @var must be a simple identifier,
 * we force a syntax error here if it isn't.
 */
#define get_cpu_var(var)						\
(*({									\
	preempt_disable();						\
	this_cpu_ptr(&var);						\
}))
```

将替换为 this_cpu_ptr 调用后，问题解决。

## get_cpu_var 和 this_cpu_ptr

| 项目 | get_cpu_var() | this_cpu_ptr() |
| ------ | ------ | ------ |
| 是否禁用抢占 | ✅ 自动禁用抢占 | ❌ 不禁用（你必须自己处理） |
| 适用上下文 | 普通线程上下文、驱动等 | 中断、softirq、已禁抢占的上下文 |
| 是否会影响调度 | ✅ 会影响调度（抢占被关闭） | ❌ 不影响调度（你负责上下文安全） |
| 返回什么 | 变量本身的引用（左值） | 指向 per-cpu 区域的指针 |
| 性能表现 | 较安全，稍慢 | 更快，但你自己负责安全 |
| 是否需要 put_cpu_var | ✅ 是的 | ❌ 不需要 |

如果只是读取 per cpu变量， 可以使用 this_cpu_ptr() 。
如果读取per cpu变量后还会操作它， 那么需要  get_cpu_var() 和 put_cpu_var() 保护起来。

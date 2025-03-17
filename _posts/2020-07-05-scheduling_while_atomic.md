---
title: "[crash analysis]BUG: scheduling while atomic"
date: 2020-07-05
---
crash的直接提示信息“Kernel panic - not syncing: Aiee, killing interrupt handler!”，不太常见。crash栈也没太多信息，COMMAND：“rmmod”提示可能是卸载模块时导致的crash。

```bash
PID: 14829  TASK: ffff88007b478000  CPU: 0   COMMAND: "rmmod"
 #0 [ffff88004a157b38] machine_kexec at ffffffff81059bab
 #1 [ffff88004a157b98] __crash_kexec at ffffffff81105812
 #2 [ffff88004a157c68] panic at ffffffff81680321
 #3 [ffff88004a157ce8] do_exit at ffffffff8108c766
 #4 [ffff88004a157d78] do_group_exit at ffffffff8108c92f
 #5 [ffff88004a157da8] get_signal_to_deliver at ffffffff8109d980
 #6 [ffff88004a157e40] do_signal at ffffffff8102a427
 #7 [ffff88004a157f30] do_notify_resume at ffffffff8102aaef
 #8 [ffff88004a157f50] retint_signal at ffffffff8168edbc
    RIP: 00007ff54a9f07f0  RSP: 00007ffcfc47fa68  RFLAGS: 00010202
    RAX: 0000000000000000  RBX: 0000000001d551f0  RCX: 0000000000000000
    RDX: 0000000000000010  RSI: 0000000001d562a0  RDI: 0000000001d56290
    RBP: 0000000001d56290   R8: 0000000000000000   R9: 000000000000003f
    R10: 00007ffcfc47f7f0  R11: 00007ff54a9f07f0  R12: 0000000001d555d8
    R13: 0000000001d555d0  R14: 0000000001d56210  R15: 0000000001d55010
    ORIG_RAX: ffffffffffffffff  CS: 0033  SS: 002b
```

查看vmcore-dmesg.txt，发现有几次“BUG: scheduling while atomic: rmmod/14829/0x00000200”打印。第一次打印如下。

```bash
[64365.793371] TESTMOD:<Warning>test server stop.
[64365.794442] BUG: scheduling while atomic: rmmod/14829/0x00000200
[64365.795672] Modules linked in: tun xt_nat veth testmod(OE-) xt_conntrack ipt_MASQUERADE nf_nat_masquerade_ipv4 nf_conntrack_netlink nfnetlink xt_addrtype iptable_filter iptable_nat nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat_ipv4 nf_nat nf_conntrack br_netfilter bridge stp llc xfrm4_tunnel tunnel4 ipcomp xfrm_ipcomp esp4 ah4 af_key l2tp_ppp l2tp_netlink l2tp_core pppox ppp_generic slhc loop nls_utf8 isofs overlay(T) iosf_mbi crc32_pclmul ghash_clmulni_intel aesni_intel lrw gf128mul glue_helper ablk_helper cryptd ppdev sg pcspkr parport_pc virtio_balloon parport i2c_piix4 ip_tables ext4 mbcache jbd2 virtio_net virtio_blk sr_mod cdrom ata_generic pata_acpi cirrus drm_kms_helper syscopyarea sysfillrect sysimgblt fb_sys_fops crct10dif_pclmul crct10dif_common ata_piix ttm crc32c_intel serio_raw virtio_pci
[64365.807828]  virtio_ring virtio drm i2c_core libata floppy
[64365.809068] CPU: 0 PID: 14829 Comm: rmmod Tainted: G           OE  ------------ T 3.10.0-514.21.1.el7.x86_64 #1
[64365.810828] Hardware name: Smdbmds KVM, BIOS seabios-1.9.1-qemu-project.org 04/01/2014
[64365.812446]  0000000000000000 0000000057419786 ffff88004a157d10 ffffffff81686f13
[64365.814052]  ffff88004a157d20 ffffffff81680bb1 ffff88004a157d80 ffffffff8168c4be
[64365.815672]  ffff88007b478000 ffff88004a157fd8 ffff88004a157fd8 ffff88004a157fd8
[64365.817251] Call Trace:
[64365.818307]  [<ffffffff81686f13>] dump_stack+0x19/0x1b
[64365.819686]  [<ffffffff81680bb1>] __schedule_bug+0x4d/0x5b
[64365.821051]  [<ffffffff8168c4be>] __schedule+0x89e/0x990
[64365.822467]  [<ffffffff8168c5d9>] schedule+0x29/0x70
[64365.823860]  [<ffffffff8168a019>] schedule_timeout+0x239/0x2c0
[64365.825267]  [<ffffffff810c1f65>] ? check_preempt_curr+0x85/0xa0
[64365.826749]  [<ffffffff810c1f99>] ? ttwu_do_wakeup+0x19/0xd0
[64365.828210]  [<ffffffff8168c9b6>] wait_for_completion+0x116/0x170
[64365.829699]  [<ffffffff810c54e0>] ? wake_up_state+0x20/0x20
[64365.831120]  [<ffffffff810b0cba>] kthread_stop+0x4a/0xe0
[64365.832564]  [<ffffffffa045d10b>] TestMode_ServerExit+0x1b/0x90 [testmod]
[64365.834120]  [<ffffffffa03e1a98>] cleanup_module+0x88/0xc0 [testmod]
[64365.835622]  [<ffffffff810fe3db>] SyS_delete_module+0x16b/0x2d0
[64365.837109]  [<ffffffff8169250b>] ? do_async_page_fault+0x1b/0xd0
[64365.838605]  [<ffffffff816975c9>] system_call_fastpath+0x16/0x1b
```

看起来是在schedule_debug函数中in_atomic_preempt_off()不为0，也就是禁止抢占情况下，执行了调度导致。

```c
static inline void schedule_debug(struct task_struct *prev)
{
 /*
  * Test if we are atomic. Since do_exit() needs to call into
  * schedule() atomically, we ignore that path for now.
  * Otherwise, whine if we are scheduling when we should not be.
  */
 if (unlikely(in_atomic_preempt_off() && !prev->exit_state))
  __schedule_bug(prev);
  rcu_sleep_check();
  
  profile_hit(SCHED_PROFILING, __builtin_return_address(0));
  
  schedstat_inc(this_rq(), sched_count);
}

```

rmmod/14829/0x00000200 中的 0x00000200就是进程的preempt_count()。

```c
#define PREEMPT_BITS 8
#define SOFTIRQ_BITS 8
#define NMI_BITS 1

//...省略部分
#define PREEMPT_SHIFT 0
#define SOFTIRQ_SHIFT (PREEMPT_SHIFT + PREEMPT_BITS)
#define HARDIRQ_SHIFT (SOFTIRQ_SHIFT + SOFTIRQ_BITS)
#define NMI_SHIFT (HARDIRQ_SHIFT + HARDIRQ_BITS)
```
对比bit位，可知进程的softirq 第2bit位置位。因为rmmod是普通进程，只能是在某个时刻禁止过软中断，才会导致preempt_count()的softirq bit位置位。

```c
#define SOFTIRQ_OFFSET (1UL << SOFTIRQ_SHIFT)	//0x100
#define SOFTIRQ_DISABLE_OFFSET (2 * SOFTIRQ_OFFSET)	//0x200

void local_bh_disable(void)
{
 __local_bh_disable(_RET_IP_, SOFTIRQ_DISABLE_OFFSET);
}

static inline void __local_bh_disable(unsigned long ip, unsigned int cnt)
{
 add_preempt_count(cnt);
 barrier();
}

# define add_preempt_count(val) do { preempt_count() += (val); } while (0)
```

也就是说在调度之前，曾经调用过 local_bh_disable()禁止中断，但是在调度时没有开启中断。

检查卸载模块相关代码，在“TESTMOD:<Warning>test server stop.”之后执行了spin_lock_bh操作(随后调用local_bh_disable)，但是没有执行spin_unlock_bh。导致调度时出现 “BUG: scheduling while atomic”。

PS：
1. __do_softirq()中会调用__local_bh_disable(_RET_IP_, SOFTIRQ_OFFSET)，preempt_count()的softirq第1个bit置位，即0x100。
2. local_bh_disable()中会调用__local_bh_disable(_RET_IP_, SOFTIRQ_DISABLE_OFFSET)，preempt_count()的softirq第2个bit置位，即0x200。所有带_bh后缀的锁，都会调用local_bh_disable()。





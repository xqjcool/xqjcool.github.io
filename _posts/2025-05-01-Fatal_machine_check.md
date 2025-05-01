---
title: "[CRASH] Kernel panic - not syncing: Fatal machine check"
date: 2025-05-01
---

# [CRASH] Kernel panic - not syncing: Fatal machine check"

客户上报设备异常重启，查看发现是出现硬件错误。第一次遇到这种情况，特此记录。

## 问题现象

- 系统日志：

```bash
[1681437.836649] mce: [Hardware Error]: CPU 0: Machine Check Exception: 5 Bank 10: fe002984001000c1
[1681437.941855] mce: [Hardware Error]: RIP !INEXACT! 10:<ffffffff80a97fa4> 
[1681437.941859] {mwait_idle+0x84/0x180}
[1681438.066860] mce: [Hardware Error]: TSC 1574f973c6b744 ADDR 9562dc000 MISC 1229440404003e8c 
[1681438.168887] mce: [Hardware Error]: PROCESSOR 0:406f1 TIME 1745767392 SOCKET 0 APIC 0 microcode b000014
[1681438.282349] mce: [Hardware Error]: Run the above through 'mcelog --ascii'
[1681438.365654] mce: [Hardware Error]: Machine check: Processor context corrupt
[1681438.451037] Kernel panic - not syncing: Fatal machine check
```

- crash调试信息：

```bash
crash> sys
      KERNEL: root/vmlinux  [TAINTED]
    DUMPFILE: coredump-2025-04-27-15_23  [PARTIAL DUMP]
        CPUS: 12
        DATE: Sun Apr 27 08:23:13 PDT 2025
      UPTIME: 19 days, 11:03:58
LOAD AVERAGE: 0.05, 0.09, 0.09
       TASKS: 381
    NODENAME: MODLE-02
     RELEASE: 4.14
     VERSION: #1 SMP Fri Jan 12 17:35:06 PST 2024
     MACHINE: x86_64  (3591 Mhz)
      MEMORY: 63.9 GB
       PANIC: "Kernel panic - not syncing: Fatal machine check"
crash> bt
PID: 0        TASK: ffff889038ef3700  CPU: 11   COMMAND: "swapper/11"
 #0 [fffffe000022bc80] machine_kexec at ffffffff8023e1d9
 #1 [fffffe000022bcd0] __crash_kexec at ffffffff802cb72b
 #2 [fffffe000022be08] mce_panic.cold at ffffffff80a5de51
 #3 [fffffe000022be40] do_machine_check at ffffffff8022f2c5
 #4 [fffffe000022bf40] do_mce at ffffffff8022f375
 #5 [fffffe000022bf50] machine_check at ffffffff80c0135b
    [exception RIP: mwait_idle+132]
    RIP: ffffffff80a97fa4  RSP: ffffc9000012beb8  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: ffff889038ef3700  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: ffffc9000012bec8   R8: 0000000000000400   R9: ffff88903f5c8940
    R10: 00000000000003a7  R11: 0000000000000000  R12: 000000000000000b
    R13: 0000000000000000  R14: 0000000000000000  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <MCE exception stack> ---
 #6 [ffffc9000012beb8] mwait_idle at ffffffff80a97fa4
 #7 [ffffc9000012bed0] arch_cpu_idle at ffffffff80222c95
 #8 [ffffc9000012bee0] default_idle_call at ffffffff80a982b3
 #9 [ffffc9000012bef0] do_idle at ffffffff8028ad31
#10 [ffffc9000012bf18] cpu_startup_entry at ffffffff8028aed0
#11 [ffffc9000012bf30] start_secondary at ffffffff80235a44
#12 [ffffc9000012bf50] secondary_startup_64 at ffffffff802000d5

```

- 其他core情况

```bash
crash> bt -c 0
PID: 0        TASK: ffffffff8141b4c0  CPU: 0    COMMAND: "swapper/0"
 #0 [fffffe000000ae58] crash_nmi_callback at ffffffff80234255
 #1 [fffffe000000ae68] nmi_handle at ffffffff8021c4d2
 #2 [fffffe000000aec0] do_nmi at ffffffff8021c6c6
 #3 [fffffe000000aef0] end_repeat_nmi at ffffffff80c01702
    [exception RIP: delay_tsc+39]
    RIP: ffffffff80a5a187  RSP: fffffe0000010e00  RFLAGS: 00000246
    RAX: 000000000607a1cc  RBX: 0000000000421b4e  RCX: 001574fa06079d1c
    RDX: 00000000001574fa  RSI: 0000000000000e08  RDI: 0000000000000000
    RBP: fffffe0000010e00   R8: 0000000000000400   R9: 0000000000000001
    R10: 0000000000000040  R11: 0000000000000036  R12: fffffe0000010f58
    R13: 0000000000000001  R14: 0000000000000015  R15: 000000000000000c
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
 #4 [fffffe0000010e00] delay_tsc at ffffffff80a5a187
 #5 [fffffe0000010e08] __const_udelay at ffffffff80a5a0d8
 #6 [fffffe0000010e18] wait_for_panic at ffffffff8022cfea
 #7 [fffffe0000010e30] mce_timed_out at ffffffff8022d61b
 #8 [fffffe0000010e40] do_machine_check at ffffffff8022ec72
 #9 [fffffe0000010f40] do_mce at ffffffff8022f375
#10 [fffffe0000010f50] machine_check at ffffffff80c0135b
    [exception RIP: mwait_idle+132]
    RIP: ffffffff80a97fa4  RSP: ffffffff81403e78  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: ffffffff8141b4c0  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: ffffffff81403e88   R8: 0000000000015ab9   R9: ffff88903f048940
    R10: 0000000000000000  R11: 0000000000000000  R12: 0000000000000000
    R13: 0000000000000000  R14: 0000000000000001  R15: 0000000000000008
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <MCE exception stack> ---
#11 [ffffffff81403e78] mwait_idle at ffffffff80a97fa4
#12 [ffffffff81403e90] arch_cpu_idle at ffffffff80222c95
#13 [ffffffff81403ea0] default_idle_call at ffffffff80a982b3
#14 [ffffffff81403eb0] do_idle at ffffffff8028ad31
#15 [ffffffff81403ed8] cpu_startup_entry at ffffffff8028aed0
#16 [ffffffff81403ef0] rest_init at ffffffff80a922c4
#17 [ffffffff81403f00] start_kernel at ffffffff815c4e0a
#18 [ffffffff81403f28] x86_64_start_reservations at ffffffff815c43aa
#19 [ffffffff81403f38] x86_64_start_kernel at ffffffff815c4430
#20 [ffffffff81403f50] secondary_startup_64 at ffffffff802000d5
crash> 
crash> 
crash> bt -c 3
PID: 0        TASK: ffff889038ec44c0  CPU: 3    COMMAND: "swapper/3"
 #0 [fffffe000009de58] crash_nmi_callback at ffffffff80234255
 #1 [fffffe000009de68] nmi_handle at ffffffff8021c4d2
 #2 [fffffe000009dec0] do_nmi at ffffffff8021c6c6
 #3 [fffffe000009def0] end_repeat_nmi at ffffffff80c01702
    [exception RIP: delay_tsc+66]
    RIP: ffffffff80a5a1a2  RSP: fffffe00000a3e00  RFLAGS: 00000287
    RAX: 001574fa06181714  RBX: 0000000000421add  RCX: 001574fa061816bc
    RDX: 0000000000000003  RSI: 0000000000000e08  RDI: 0000000000000003
    RBP: fffffe00000a3e00   R8: 0000000000000004   R9: 0000000000000001
    R10: 0000000000000040  R11: 000000000000003e  R12: fffffe00000a3f58
    R13: 0000000000000001  R14: 0000000000000015  R15: 000000000000000c
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
 #4 [fffffe00000a3e00] delay_tsc at ffffffff80a5a1a2
 #5 [fffffe00000a3e08] __const_udelay at ffffffff80a5a0d8
 #6 [fffffe00000a3e18] wait_for_panic at ffffffff8022cfea
 #7 [fffffe00000a3e30] mce_timed_out at ffffffff8022d61b
 #8 [fffffe00000a3e40] do_machine_check at ffffffff8022ec72
 #9 [fffffe00000a3f40] do_mce at ffffffff8022f375
#10 [fffffe00000a3f50] machine_check at ffffffff80c0135b
    [exception RIP: mwait_idle+132]
    RIP: ffffffff80a97fa4  RSP: ffffc900000ebeb8  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: ffff889038ec44c0  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: ffffc900000ebec8   R8: 0000000000000000   R9: ffff88903f1c8940
    R10: 0000000000000084  R11: 0000000000000000  R12: 0000000000000003
    R13: 0000000000000000  R14: 0000000000000000  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <MCE exception stack> ---
#11 [ffffc900000ebeb8] mwait_idle at ffffffff80a97fa4
#12 [ffffc900000ebed0] arch_cpu_idle at ffffffff80222c95
#13 [ffffc900000ebee0] default_idle_call at ffffffff80a982b3
#14 [ffffc900000ebef0] do_idle at ffffffff8028ad31
#15 [ffffc900000ebf18] cpu_startup_entry at ffffffff8028aed0
#16 [ffffc900000ebf30] start_secondary at ffffffff80235a44
#17 [ffffc900000ebf50] secondary_startup_64 at ffffffff802000d5

```

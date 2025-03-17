---
title: "[crash analysis]Kernel panic - not syncing: Aiee, killing interrupt handler!"
date: 2020-08-06
---
公司产品升级后，测试发现下发某个配置失败时会导致系统crash。

crash查看栈显示如下，没有有用信息。
```
crash> bt
PID: 3348   TASK: ffff880175034e70  CPU: 3   COMMAND: "test.sh"
 #0 [ffff88016c063b38] machine_kexec at ffffffff81059beb
 #1 [ffff88016c063b98] __crash_kexec at ffffffff81105822
 #2 [ffff88016c063c68] panic at ffffffff81680541
 #3 [ffff88016c063ce8] do_exit at ffffffff8108c766
 #4 [ffff88016c063d78] do_group_exit at ffffffff8108c92f
 #5 [ffff88016c063da8] get_signal_to_deliver at ffffffff8109d980
 #6 [ffff88016c063e40] do_signal at ffffffff8102a427
 #7 [ffff88016c063f30] do_notify_resume at ffffffff8102aaef
 #8 [ffff88016c063f50] retint_signal at ffffffff8168effc
    RIP: 00007fe2414599f0  RSP: 00007fff17ea1938  RFLAGS: 00010246
    RAX: 00007fe241e306c0  RBX: 0000000000000016  RCX: 0000000000000000
    RDX: 0000000000000000  RSI: 00007fe2415a67fd  RDI: 00007fe2415a643c
    RBP: 0000000000000000   R8: 0000000000000000   R9: 0000000000000005
    R10: 00007fff17ea1740  R11: 00007fe2414b0910  R12: 0000000000000000
    R13: 0000000000000000  R14: 0000000000000001  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0033  SS: 002b
```

查看vmcore-dmesg发现有 scheduling while atomic: test.sh/3348/0x00000400。

```
[  287.366523] BUG: scheduling while atomic: test.sh/3348/0x00000400
[  287.366525] Modules linked in: lwconn(OE) bridge stp llc sha512_ssse3 sha512_generic intel_powerclamp drbg ansi_cprng coretemp iosf_mbi kvm_intel igb(OE) kvm irqbypass crc32_pclmul iTCO_wdt ghash_clmulni_intel iTCO_vendor_support aesni_intel lrw ipmi_ssif qat_c3xxx gf128mul glue_helper ablk_helper pcspkr intel_qat cryptd i2c_i801 i2c_ismt authenc ipmi_msghandler i2c_algo_bit i2c_core shpchp ip_tables xfs libcrc32c mmc_block ahci libahci crct10dif_pclmul sdhci_pci ixgbe(OE) crct10dif_common sdhci dca crc32c_intel ptp pps_core mmc_core libata fjes dm_mirror dm_region_hash dm_log dm_mod nf_conntrack [last unloaded: lwconn]
[  287.366571] CPU: 3 PID: 3348 Comm: link52.sh Tainted: G           OE  ------------   3.10.0-514.26.2.el7.x86_64 #1
[  287.366573] Hardware name: Default string DTA1161AC4/Default string, BIOS 5.13 (G168-025) 07/08/2020
[  287.366574]  0000000000000000 0000000075a7f477 ffff88016c063ee8 ffffffff81687133
[  287.366578]  ffff88016c063ef8 ffffffff81680dd1 ffff88016c063f58 ffffffff8168c6de
[  287.366581]  ffff880175034e70 ffff88016c063fd8 ffff88016c063fd8 ffff88016c063fd8
[  287.366584] Call Trace:
[  287.366593]  [<ffffffff81687133>] dump_stack+0x19/0x1b
[  287.366596]  [<ffffffff81680dd1>] __schedule_bug+0x4d/0x5b
[  287.366600]  [<ffffffff8168c6de>] __schedule+0x89e/0x990
[  287.366603]  [<ffffffff8168d831>] schedule_user+0x31/0xc0
[  287.366606]  [<ffffffff81697875>] sysret_careful+0x14/0x17
```

也就是说出现了在中断中调度的情况。preempt_count()为0x00000400。
之前讲过 
软中断中我们会 preempt_count()+0x100;
local_bh_disable()会 preempt_count()+0x200;

0x400看起来是 local_bh_disable() 嵌套后的结果。
从log中我们可知每次下发link配置并且失败后，会导致crash。配置下发正常却没有问题。

初步查看link配置处理代码，没有发现异常(大意了)。于是增加debug log，打印preempt_count()。
复现后，查看debug log，发现link配置处理代码执行前是preempt_count()是0x00，嵌套调用 **spin_lock_bh(A),spin_lock_bh(B)** 后，是0x400，函数异常返回后，preempt_count()依然是0x400。

再仔细查看link配置处理代码，发先正常处理中，会调用 **spin_unlock_bh(B),spin_unlock_bh(A)**。然后函数返回，一切正常。

但是在异常处理逻辑中，调用了 **spin_unlock(B),spin_unlock(A)**。lock B和A是释放了，但是中断下半部没有enable，导致函数退出后，进程处在中断下半部disable状态。一旦有调度情况，就会出现scheduling while atomic 错误，导致系统异常。

后记：
当我们发现 scheduling while atomic 后，查看 preempt_count()，根据preempt_count()的值，来分析相关操作中对应的lock/unlock是否不匹配。

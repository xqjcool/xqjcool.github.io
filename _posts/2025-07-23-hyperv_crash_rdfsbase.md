---
title: "Crash analysis of Hyper-V virtual machine startup"
date: 2025-07-23
---

# Hyper-V虚拟机启动crash分析

## 1. 问题现象

测试上报说Hyper-V虚拟机因为crash导致无法正常启动。串口异常栈信息如下

<img width="1286" height="801" alt="image" src="https://github.com/user-attachments/assets/f402ca00-05cd-4c77-a395-defb9afa939a" />

其他虚拟机平台和实体机都正常。

## 2. 初步分析

看到系统crash在 copy_thread+0xd2位置，于是 `gdb vmlinux` ，查看对应位置。

```bash
(gdb) disass copy_thread
Dump of assembler code for function copy_thread:
   0xffffffff8025c530 <+0>:	nopl   0x0(%rax,%rax,1)
   0xffffffff8025c535 <+5>:	push   %rbp
//省略无关
   0xffffffff8025c5d9 <+169>:	callq  *0x1bb8f59(%rip)        # 0xffffffff81e15538 <pv_ops+248>
   0xffffffff8025c5df <+175>:	mov    %gs:0x7fdf32d9(%rip),%r15        # 0x4f8c0 <current_task>
   0xffffffff8025c5e7 <+183>:	mov    %fs,%ax
   0xffffffff8025c5ea <+186>:	mov    %ax,0xb64(%r15)
   0xffffffff8025c5f2 <+194>:	mov    %gs,%ax
   0xffffffff8025c5f5 <+197>:	mov    %ax,0xb66(%r15)
   0xffffffff8025c5fd <+205>:	jmpq   0xffffffff81f60a3e
   0xffffffff8025c602 <+210>:	rdfsbase %rax        //这里  0xd2对应210
   0xffffffff8025c607 <+215>:	mov    %rax,0xb68(%r15)
   0xffffffff8025c60e <+222>:	callq  0xffffffff810405e0 <__rdgsbase_inactive>
```

看起来像是不支持rdfsbase指令导致的 `invalid opcode` 异常。 

这部分对应函数 save_fsgs， 查看代码：

```c
static __always_inline void save_fsgs(struct task_struct *task)
{
	savesegment(fs, task->thread.fsindex);
	savesegment(gs, task->thread.gsindex);
	if (static_cpu_has(X86_FEATURE_FSGSBASE)) {    //支持FSGSBASE 则使用 rdfsbase指令
		/*
		 * If FSGSBASE is enabled, we can't make any useful guesses
		 * about the base, and user code expects us to save the current
		 * value.  Fortunately, reading the base directly is efficient.
		 */
		task->thread.fsbase = rdfsbase();
		task->thread.gsbase = __rdgsbase_inactive();
	} else {                                      //不支持则使用旧指令
		save_base_legacy(task, task->thread.fsindex, FS);
		save_base_legacy(task, task->thread.gsindex, GS);
	}
}
```

当CPU 支持FSGSBASE功能时，才会使用rdfsbase指令， 也就是说当前CPU是支持的。
从异常栈CR4寄存器看也能确认。 
CR4=0x1506f0 ,第16个bit为1，表示 CPU 支持FSGSBASE。

```c
#define X86_CR4_FSGSBASE_BIT	16 /* enable RDWRFSGS support */
#define X86_CR4_FSGSBASE	_BITUL(X86_CR4_FSGSBASE_BIT)

static void identify_cpu(struct cpuinfo_x86 *c)
{
	int i;

	c->loops_per_jiffy = loops_per_jiffy;
	c->x86_cache_size = 0;
//忽略无关

	/* Enable FSGSBASE instructions if available. */
	if (cpu_has(c, X86_FEATURE_FSGSBASE)) {
		cr4_set_bits(X86_CR4_FSGSBASE);    //set CR4 bit16
		elf_hwcap2 |= HWCAP2_FSGSBASE;
	}
//忽略无关

}
```

这就有点诡异了， 明明CPU支持，但在运行时又认为时invalid opcode。

## 3. 继续深入

为了印证猜想，并找出真相，做了以下验证。

### 3.1 关闭CPU FSGSBASE

在启动参数里加入 nofsgsbase ,关闭CPU对fsgs的支持，

```c
static __init int x86_nofsgsbase_setup(char *arg)
{
	/* Require an exact match without trailing characters. */
	if (strlen(arg))
		return 0;

	/* Do not emit a message if the feature is not present. */
	if (!boot_cpu_has(X86_FEATURE_FSGSBASE))
		return 1;

	setup_clear_cpu_cap(X86_FEATURE_FSGSBASE);
	pr_info("FSGSBASE disabled via kernel command line\n");
	return 1;
}
__setup("nofsgsbase", x86_nofsgsbase_setup);
```

修改后，系统可以正常启动。 也印证了就是CPU不支持rdfsbase指令导致的。

### 3.2 在宿主机上创建B产品的Hyper-V虚拟机

公司还有B产品，内核和该产品相同，且同样有Hyper-V虚拟机，但是没听说QA上报Hyper-V虚拟机启动异常问题。
联系B产品QA，提供了一台Hyper-V虚拟机，登陆后确实工作正常。

为了对比差异，在问题宿主机上手动创建了B产品的Hyper-V虚拟机， 结果同样出现了启动后crash问题。
于是怀疑问题宿主机有问题，对fsgsbase的指令支持异常。

为了确认猜想，在B产品的测试宿主机上创建了A产品的Hyper-V虚拟机，结果运行正常。
也就是说同样的A产品image，在问题宿主机上无法启动，在B产品的宿主机上就能正常运行。

### 3.3 对比两台宿主机CPU差异

- 先是在问题宿主机上，部署一台升级内核前的Hyper-V虚拟机，因为旧内核没有支持fsgsbase指令，所以正常运行。
查看虚拟机CPU：

```bash
/var/log# cat /proc/cpuinfo 
processor	: 0
vendor_id	: AuthenticAMD
cpu family	: 23
model		: 1
model name	: AMD EPYC 7281 16-Core Processor
stepping	: 2
microcode	: 0xffffffff
cpu MHz		: 2099.995
cache size	: 512 KB
physical id	: 0
siblings	: 2
core id		: 0
cpu cores	: 2
apicid		: 0
initial apicid	: 0
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt lm rep_good nopl cpuid extd_apicid pni pclmulqdq ssse3 fma cx16 sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand hypervisor lahf_lm cmp_legacy cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw ssbd vmmcall fsgsbase bmi1 avx2 smep bmi2 xsaveopt arat
bugs		: fxsave_leak sysret_ss_attrs null_seg spectre_v1 spectre_v2 spec_store_bypass
bogomips	: 4199.99
TLB size	: 2560 4K pages
clflush size	: 64
cache_alignment	: 64
address sizes	: 42 bits physical, 48 bits virtual
power management:
```

是 `AMD EPYC 7281 16-Core Processor` 且支持 fsgsbase指令集

- 随后在可以正常运行的新内核Hyper-V虚拟机上查看

```bash
/var/log# cat /proc/cpuinfo 
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 63
model name	: Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
stepping	: 2
microcode	: 0xffffffff
cpu MHz		: 2399.978
cache size	: 15360 KB
physical id	: 0
siblings	: 1
core id		: 0
cpu cores	: 1
apicid		: 0
initial apicid	: 0
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 movbe popcnt xsave avx f16c rdrand hypervisor lahf_lm abm invpcid_single ibrs ibpb stibp fsgsbase bmi1 avx2 smep bmi2 erms invpcid xsaveopt arch_capabilities
bugs		: cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs itlb_multihit mmio_stale_data retbleed
bogomips	: 4799.95
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:
```

是 `Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz` 同样也支持 fsgsbase指令集。

## 4. 如何修正问题宿主机

最终问题出在这个有问题的宿主机，它明明支持fsgsbase指令集，但是运行时却又invalid opcode。
如果解决这种问题？经过研究，我们可以在创建虚拟机时，把处理器Processor的兼容性compatibility选项选中。

<img width="724" height="258" alt="image" src="https://github.com/user-attachments/assets/b6a07e22-e77e-4baf-8238-f8623124364d" />

打开后，经测试，Hyper-V虚拟机可以正常启动。查看发现CPU特性不再支持fsgsbase。

<img width="324" height="179" alt="image" src="https://github.com/user-attachments/assets/b8c5edc2-a8b3-4316-a878-699b85841d7a" />

## 5. 后记

[FSGSBASE Instructions](https://docs.oracle.com/cd/E37838_01/html/E61064/gnydr.html)
[Accessing FS/GS base with the FSGSBASE instructions]([https://www.kernel.org/doc/html/next/x86/x86_64/fsgs.html](https://www.kernel.org/doc/html/next/x86/x86_64/fsgs.html#accessing-fs-gs-base-with-the-fsgsbase-instructions))

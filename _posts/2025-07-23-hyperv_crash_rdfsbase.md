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

为了对比差异，在宿主机是手动创建了B产品的Hyper-V虚拟机， 结果同样出现了启动后crash问题。



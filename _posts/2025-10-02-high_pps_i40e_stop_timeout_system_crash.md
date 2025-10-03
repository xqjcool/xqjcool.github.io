---
title: "System crash caused by i40e NIC stop timeout under high PPS traffic"
date: 2025-10-02
---

# 高pps流量下i40e网卡stop超时导致的系统crash

## 1. 问题现象

最近QA在做性能测试，没有关闭流量的情况下，对测试设备进行了重启。刚重启没多久就出现了crash，重启后又crash。直到停止流量后，系统才正常启动。

当系统正常启动后，再开启流量测试，则工作正常。或者流量不够大，系统启动也不会crash。

也就是说问题的触发满足两个条件。
1. 高PPS流量
2. 系统刚启动

## 2. 初步分析

取coredump分析

```bash
       PANIC: "general protection fault: 0000 [#1] SMP NOPTI"
         PID: 2815
     COMMAND: "ifconfig"
        TASK: ffffa15a928af000  [THREAD_INFO: ffffa15a928af000]
         CPU: 5
       STATE: TASK_RUNNING (PANIC)

```

"general protection fault: 0000 [#1] SMP NOPTI" 这种错误一般都是 异常地址访问触发访问保护。我们查看对应的异常地址

```bash
crash> bt
PID: 2815     TASK: ffffa15a928af000  CPU: 5    COMMAND: "ifconfig"
 #0 [ffffa44b5bc4fba8] machine_kexec at ffffffff9743e54c
 #1 [ffffa44b5bc4fbf8] __crash_kexec at ffffffff974cbd3b
 #2 [ffffa44b5bc4fcd0] oops_end at ffffffff9741c009
 #3 [ffffa44b5bc4fcf8] die at ffffffff9741c083
 #4 [ffffa44b5bc4fd28] do_general_protection at ffffffff9741a45e
 #5 [ffffa44b5bc4fd50] general_protection at ffffffff97e01325
    [exception RIP: acct_collect+92]
    RIP: ffffffff974cb68c  RSP: ffffa44b5bc4fe08  RFLAGS: 00010202
    RAX: 004500087d7dca0b  RBX: ffffa15a92b28400  RCX: 0000000000000000
    RDX: 90003f1058ecf300  RSI: 0000000000000001  RDI: ffffa15a75a90d40
    RBP: ffffa44b5bc4fe28   R8: 0000000000000000   R9: 00007f8e150a58d8
    R10: 0000000000000000  R11: 0000000000000000  R12: 0000000000000000
    R13: 0000000000000000  R14: ffffa15a928af000  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #6 [ffffa44b5bc4fe30] do_exit at ffffffff97456837
 #7 [ffffa44b5bc4fea0] do_group_exit at ffffffff974571db
 #8 [ffffa44b5bc4fed0] sys_exit_group at ffffffff97457264
 #9 [ffffa44b5bc4fee0] do_syscall_64 at ffffffff974016d5
#10 [ffffa44b5bc4ff50] entry_SYSCALL_64_after_hwframe at ffffffff97e0008d

crash> dis -l acct_collect+92
/root/linux-4.14.336/kernel/acct.c: 547
0xffffffff974cb68c <acct_collect+92>:   add    0x8(%rax),%rdx    //访问rax寄存器

```

因为 rax = 004500087d7dca0b 不是一个正常的地址，所以导致 "general protection fault"。
那么rax是什么，从哪里来？

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/ecb57906-de2a-4b22-8c57-fbfd0147e116" />

```bash
crash> dis -r acct_collect+92
0xffffffff974cb630 <acct_collect>:      nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff974cb635 <acct_collect+5>:    push   %rbp
0xffffffff974cb636 <acct_collect+6>:    mov    %rsp,%rbp
0xffffffff974cb639 <acct_collect+9>:    push   %r14
0xffffffff974cb63b <acct_collect+11>:   push   %r13
0xffffffff974cb63d <acct_collect+13>:   mov    %gs:0x19dc0,%r14    //current指向的当前task_struct
0xffffffff974cb646 <acct_collect+22>:   push   %r12
0xffffffff974cb648 <acct_collect+24>:   mov    %rdi,%r12
0xffffffff974cb64b <acct_collect+27>:   push   %rbx
0xffffffff974cb64c <acct_collect+28>:   mov    0x650(%r14),%rbx
0xffffffff974cb653 <acct_collect+35>:   test   %esi,%esi
0xffffffff974cb655 <acct_collect+37>:   je     0xffffffff974cb7a2 <acct_collect+370>
0xffffffff974cb65b <acct_collect+43>:   xor    %r13d,%r13d
0xffffffff974cb65e <acct_collect+46>:   cmpq   $0x0,0x3a0(%r14)
0xffffffff974cb666 <acct_collect+54>:   je     0xffffffff974cb6bc <acct_collect+140>
0xffffffff974cb668 <acct_collect+56>:   mov    0x3a0(%r14),%rax
0xffffffff974cb66f <acct_collect+63>:   lea    0x80(%rax),%rdi
0xffffffff974cb676 <acct_collect+70>:   call   0xffffffff97cde3e0 <down_read>
0xffffffff974cb67b <acct_collect+75>:   mov    0x3a0(%r14),%rax  //task_struct.mm
0xffffffff974cb682 <acct_collect+82>:   mov    (%rax),%rax
0xffffffff974cb685 <acct_collect+85>:   test   %rax,%rax
0xffffffff974cb688 <acct_collect+88>:   je     0xffffffff974cb6a3 <acct_collect+115>
0xffffffff974cb68a <acct_collect+90>:   xor    %edx,%edx
0xffffffff974cb68c <acct_collect+92>:   add    0x8(%rax),%rdx

crash> task_struct.mm -ox
struct task_struct {
   [0x3a0] struct mm_struct *mm;
}
crash> mm_struct.mmap -ox
struct mm_struct {
    [0x0] struct vm_area_struct *mmap;
}

```

配合代码和反汇编，我们知道 rax是 vm_area_struct *vma指针。

```bash
crash> task_struct.mm -x ffffa15a928af000
  mm = 0xffffa15a75a90cc0,
crash> mm_struct.mmap -x 0xffffa15a75a90cc0
  mmap = 0xffffa15a4533c2e0
crash> vm_area_struct.vm_next -ox
struct vm_area_struct {
  [0x10] struct vm_area_struct *vm_next;
}
crash> list -o 0x10 0xffffa15a4533c2e0  //循环查看所有vm_area_struct
ffffa15a4533c2e0
ffffa15a4533c228
ffffa15a4533c170
ffffa15a4533c0b8
4500087d7dca0b      //出问题的vm_area_struct
list: invalid kernel virtual address: 4500087d7dca1b  type: "list entry"

```

可以看到其中 vm_area_struct ffffa15a4533c0b8 的 vm_next是个异常的值， 导致while (vma) 遍历操作访问 vma->end时出错。

```bash
crash> kmem ffffa15a4533c0b8
CACHE             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ffffa15a93632300      184       1897      4246    193     4k  vm_area_struct
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffe1932014cf00  ffffa15a4533c000     0     22         22     0
  FREE / [ALLOCATED]
  [ffffa15a4533c0b8]

      PAGE        PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffe1932014cf00 80533c000                0        0  1 8000000000000100 slab

crash> vm_area_struct -x ffffa15a4533c0b8
struct vm_area_struct {
  vm_start = 0x48c000,
  vm_end = 0x90003f10592d0300,
  vm_next = 0x4500087d7dca0b,
  vm_prev = 0x11ff0000ca1f2e00,
  vm_rb = {
    __rb_parent_color = 0x101030101012734,
    rb_right = 0x1a005000ec9ac964,
    rb_left = 0x0
  },
  rb_subtree_gap = 0x0,
  vm_mm = 0xffffa15a00000000,       //一半被写坏
  vm_page_prot = {
    pgprot = 0x8000000000000025
  },
  vm_flags = 0x100871,
  shared = {
    rb = {
      __rb_parent_color = 0xffffa15a453bb9b0,
      rb_right = 0x0,
      rb_left = 0x0
    },
    rb_subtree_last = 0x8c
  },
  anon_vma_chain = {
    next = 0xffffa15a45208c50,
    prev = 0xffffa15a45208c50
  },
  anon_vma = 0xffffa15a44847a00,
  vm_ops = 0xffffffff98227480 <ext4_file_vm_ops>,
  vm_pgoff = 0x8b,
  vm_file = 0xffffa15a45352000,
  vm_private_data = 0x0,
  swap_readahead_info = {
    counter = 0x0
  },
  vm_userfaultfd_ctx = {<No data fields>}
}
```

可以看到 ffffa15a4533c0b8 是被slab申请用作 vm_area_struct， 但是vm_mm之前的内存被写坏。

初步判定是内存被异常覆写。是UAF(use-after-free)么？谁覆写的？

## 3. 深入挖掘

当我盯着异常数据看的时候，"vm_next = 0x4500087d7dca0b"中0x45这个数字特别的醒目。直觉告诉我这可能是个IP header数据。经常分析报文的都知道IP头部第一个字节值就是0x45。
于是我用ip_hdr试着读取。

```bash
crash> rd ffffa15a4533c0b8 23  
ffffa15a4533c0b8:  000000000048c000 90003f10592d0300   ..H.......-Y.?..
ffffa15a4533c0c8:  004500087d7dca0b 11ff0000ca1f2e00   ..}}..E.........
ffffa15a4533c0d8:  0101030101012734 1a005000ec9ac964   4'......d....P..
ffffa15a4533c0e8:  0000000000000000 0000000000000000   ................
ffffa15a4533c0f8:  ffffa15a00000000 8000000000000025   ....Z...%.......
ffffa15a4533c108:  0000000000100871 ffffa15a453bb9b0   q.........;EZ...
ffffa15a4533c118:  0000000000000000 0000000000000000   ................
ffffa15a4533c128:  000000000000008c ffffa15a45208c50   ........P. EZ...
ffffa15a4533c138:  ffffa15a45208c50 ffffa15a44847a00   P. EZ....z.DZ...
ffffa15a4533c148:  ffffffff98227480 000000000000008b   .t".............
ffffa15a4533c158:  ffffa15a45352000 0000000000000000   . 5EZ...........
ffffa15a4533c168:  0000000000000000                    ........

crash> iphdr -x ffffa15a4533c0ce
struct iphdr {
  ihl = 0x5,
  version = 0x4,
  tos = 0x0,
  tot_len = 0x2e00,
  id = 0xca1f,
  frag_off = 0x0,
  ttl = 0xff,
  protocol = 0x11,       //UDP
  check = 0x2734,
  saddr = 0x3010101,       //1.1.1.3
  daddr = 0xc9640101       //1.1.100.201
}
crash> net -N 0x3010101
1.1.1.3
crash> net -N 0xc9640101
1.1.100.201

```

确实是IP/UDP报文。 也就是说某个处理中把 vm_area_struct内存当做skb的buffer操作了。

```bash
[   24.702850] 8021q: adding VLAN 0 to HW filter on device eth9
[   24.703387] IPv6: ADDRCONF(NETDEV_CHANGE): eth9: link becomes ready
[   24.761472] i40e 0000:07:00.3: VSI seid 399 Rx ring 0 disable timeout
[   24.839959] i40e 0000:07:00.3 eth9: NIC Link is Down
[   24.845870] 8021q: adding VLAN 0 to HW filter on device eth10
[   24.906828] i40e 0000:07:00.2: VSI seid 398 Rx ring 0 disable timeout
[   24.985357] i40e 0000:07:00.2 eth10: NIC Link is Down
[   24.991310] 8021q: adding VLAN 0 to HW filter on device eth11
[   24.991814] general protection fault: 0000 [#1] SMP NOPTI

```

crash之前有几个接口 up/down操作。eth9和eth10都出现了 "Rx ring disable timeout" 。
查看代码这是

```c
static int i40e_vsi_control_rx(struct i40e_vsi *vsi, bool enable)
{
	struct i40e_pf *pf = vsi->back;
	int i, pf_q, ret = 0;

	pf_q = vsi->base_queue;
	for (i = 0; i < vsi->num_queue_pairs; i++, pf_q++) {
		ret = i40e_control_wait_rx_q(pf, pf_q, enable);
		if (ret) {
			dev_info(&pf->pdev->dev,
				 "VSI seid %d Rx ring %d %sable timeout\n",              //这里打印
				 vsi->seid, pf_q, (enable ? "en" : "dis"));
			break;
		}
	}

	/* HW needs up to 50ms to finish RX queue disable*/
	if (!enable)
		mdelay(50);

	return ret;
}

int i40e_control_wait_rx_q(struct i40e_pf *pf, int pf_q, bool enable)
{
	int ret = 0;

	i40e_control_rx_q(pf, pf_q, enable);

	/* wait for the change to finish */
	ret = i40e_pf_rxq_wait(pf, pf_q, enable);       //这里返回错误
	if (ret)
		return ret;

	return ret;
}

static void i40e_control_rx_q(struct i40e_pf *pf, int pf_q, bool enable)
{
	struct i40e_hw *hw = &pf->hw;
	u32 rx_reg;
	int i;

	for (i = 0; i < I40E_QTX_ENA_WAIT_COUNT; i++) {
		rx_reg = rd32(hw, I40E_QRX_ENA(pf_q));
		if (((rx_reg >> I40E_QRX_ENA_QENA_REQ_SHIFT) & 1) ==
		    ((rx_reg >> I40E_QRX_ENA_QENA_STAT_SHIFT) & 1))
			break;
		usleep_range(1000, 2000);
	}

	/* Skip if the queue is already in the requested state */
	if (enable == !!(rx_reg & I40E_QRX_ENA_QENA_STAT_MASK))
		return;

	/* turn on/off the queue */
	if (enable)
		rx_reg |= I40E_QRX_ENA_QENA_REQ_MASK;
	else
		rx_reg &= ~I40E_QRX_ENA_QENA_REQ_MASK;

	wr32(hw, I40E_QRX_ENA(pf_q), rx_reg);              //disable rx queue
}

static int i40e_pf_rxq_wait(struct i40e_pf *pf, int pf_q, bool enable)
{
	int i;
	u32 rx_reg;

	for (i = 0; i < I40E_QUEUE_WAIT_RETRY_LIMIT; i++) {       //循环I40E_QUEUE_WAIT_RETRY_LIMIT = 10 次
		rx_reg = rd32(&pf->hw, I40E_QRX_ENA(pf_q));       //读取寄存器
		if (enable == !!(rx_reg & I40E_QRX_ENA_QENA_STAT_MASK))       //下发成功则跳出
			break;

		usleep_range(10, 20);       //每次等待10~20微秒
	}
	if (i >= I40E_QUEUE_WAIT_RETRY_LIMIT)
		return -ETIMEDOUT;       //超时返回错误

	return 0;
}
```

这个错误log表示 接收DMA队列关闭失败，即DMA仍在工作。随后的操作会释放DMA内存。

```bash
i40e_down --> i40e_vsi_stop_rings --> i40e_vsi_control_rx --> i40e_control_rx_q	//停止DMA队列
          |--> i40e_clean_rx_ring --> 						//释放DMA内存
```

这样会造成 DMA队列还在工作，但是DMA内存可能已经被其他地方申请使用了。

## 4. 解决方案

### 4.1 方案1

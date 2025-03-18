---
title: "[crash analysis][mips]CPU 0 Unable to handle kernel paging request at virtual address -2"
date: 2020-03-03
---

openwrt定制低端设备，加载业务模块后很快就crash，之前帖子说过配置太低，所以系统异常crash后无法保存coredump，只有串口记录crash log供分析定位。
crash log如下：
```powershell
[   39.985393] CPU 0 Unable to handle kernel paging request at virtual address 78ddc304, epc == 80260e64, ra == 80260de4
[   39.996395] Oops[#1]:
[   39.998754] CPU: 0 PID: 3 Comm: ksoftirqd/0 Tainted: G        W       4.4.59 #0
[   40.006303] task: 87c34a10 ti: 87c44000 task.ti: 87c44000
[   40.011878] $ 0   : 00000000 00000001 78ddc304 00000001
[   40.017308] $ 4   : 87690c48 87690108 00000000 81102390
[   40.022738] $ 8   : 00000000 00000000 00000000 00000000
[   40.028167] $12   : 00000000 0a010801 00000000 0a010801
[   40.033597] $16   : 876900c0 87690c00 00000001 87323e18
[   40.039027] $20   : 0000fec0 00000052 80430000 876900c0
[   40.044456] $24   : 00000000 803f5e80                  
[   40.049886] $28   : 87c44000 87c459a0 85ab94c8 80260de4
[   40.055317] Hi    : 0000000b
[   40.058292] Lo    : 3db83e48
[   40.061300] epc   : 80260e64 __copy_skb_header+0xc0/0xfc
[   40.066795] ra    : 80260de4 __copy_skb_header+0x40/0xfc
[   40.072277] Status: 1000fc03 KERNEL EXL IE 
[   40.076625] Cause : 00800008 (ExcCode 02)
[   40.080763] BadVA : 78ddc304
[   40.083739] PrId  : 00019374 (MIPS 24Kc)
...//省略部分
[   40.164913] Process ksoftirqd/0 (pid: 3, threadinfo=87c44000, task=87c34a10, tls=00000000)
[   40.173444] Stack : 00000000 00000000 00000000 87323e28 ffff9ad0 87690c00 876900c0 80260f28
          00000000 0a010801 00000000 00000001 00000010 87323e00 876900c0 8027ce9c
          8708e000 8027f8b0 00000001 8708e000 00000000 87323e00 00000001 8027fb24
          87cd9d00 8027f61c 00000024 cdf30370 87c45ad8 876900c0 87323e00 87fad300
          8708e000 80280580 00000000 80420000 00000000 876900c0 87fad300 8708e000
          ...
[   40.210434] Call Trace:
[   40.212968] [<80260e64>] __copy_skb_header+0xc0/0xfc
[   40.218107] [<80260f28>] __skb_clone+0x28/0x100
[   40.222811] [<8027ce9c>] neigh_probe+0x34/0x80
[   40.227407] [<8027fb24>] __neigh_event_send+0x240/0x2ac
[   40.232806] [<80280580>] neigh_resolve_output+0x70/0x1b8
```

先粗略过一下异常发生时的log，错误原因是异常地址访问，Call Trace显示出错函数是__copy_skb_header，出错在内核函数调用中。内核函数一般不会有漏洞，大概率是传入的参数结构有问题。
我们将__copy_skb_header反汇编，找到具体的出错指令。由__copy_skb_header+0xc0/0xfc可知出错位置是 0xc0 ll v1,0(v0)，这是load linked指令，一般和sc(store conditional)指令配合使用，用于原子操作。

```powershell
00000000 <__copy_skb_header>:
   0: 27bdffe0  addiu sp,sp,-32
   4: afb10018  sw s1,24(sp)
   8: afb00014  sw s0,20(sp)
   c: afbf001c  sw ra,28(sp)
  10: 8ca3000c  lw v1,12(a1)
  14: 8ca20008  lw v0,8(a1)
  18: 00a08025  move s0,a1	//s0=a1
  1c: ac83000c  sw v1,12(a0)
  20: ac820008  sw v0,8(a0)
  24: 8ca20014  lw v0,20(a1)
  28: 00808825  move s1,a0
  ...//省略部分
  ac: 8e020054  lw v0,84(s0)
  b0: ae220054  sw v0,84(s1)
  b4: 8e020054  lw v0,84(s0)	//v0=[s0 + 84]
  b8: 10400006  beqz v0,d4 <__copy_skb_header+0xd4>
  bc: 00000000  nop
  c0: c0430000  ll v1,0(v0)  //出错指令
  c4: 24630001  addiu v1,v1,1
  c8: e0430000  sc v1,0(v0)
  cc: 1060fffc  beqz v1,c0 <__copy_skb_header+0xc0>
  d0: 00000000  nop
  d4: 96020064  lhu v0,100(s0)
  d8: 8fbf001c  lw ra,28(sp)
  dc: 26050068  addiu a1,s0,104
  e0: a6220064  sh v0,100(s1)
  e4: 8fb00014  lw s0,20(sp)
  e8: 26240068  addiu a0,s1,104
  ec: 8fb10018  lw s1,24(sp)
  f0: 24060038  li a2,56
  f4: 08000000  j 0 <__copy_skb_header>
  f8: 27bd0020  addiu sp,sp,32
```
读取v0指向的内存出错，v0对应\$3寄存器，可以看到v0寄存器值为78ddc304。和错误提示中virtual address 78ddc304一致。
那么这个v0保存的是什么值呢，我们向前查看，可知v0=[s0 + 84]，再向前查找，发现s0=a1，a1保存的是函数第二个入参，即struct sk_buff *old。那么v0就是skb偏移84个字节指向的结构变量。查看sk_buff结构定义，可知offset 84指向struct nf_conntrack *nfct;
到这里逻辑应该比较清晰了，再看__copy_skb_header函数，在__nf_copy(new, old, false);中调用nf_conntrack_get函数传入nfct地址，在访问nfct的use变量时，因为nfct地址异常，导致crash。

```c
//sk_buff结构
struct sk_buff {                                                              
    union {                                                                   
        struct {                                                              
            /* These two members must be first. */                            
            struct sk_buff      *next;                                        
            struct sk_buff      *prev;                                        
                                                                              
            union {                                                           
                ktime_t     tstamp;                                           
                struct skb_mstamp skb_mstamp;                                 
            };                                                                
        };                                                                    
        struct rb_node  rbnode; /* used in netem & tcp stack */               
    };                                                                        
    struct sock     *sk;                                                      
    struct net_device   *dev;                                                 
                                                                              
    /*                                                                        
     * This is the control buffer. It is free to use for every                
     * layer. Please put your private variables there. If you                 
     * want to keep them across layers you have to do a skb_clone()           
     * first. This is owned by whoever has the skb queued ATM.                
     */                                                                       
    char            cb[48] __aligned(8);                                      
                                                                              
    unsigned long       _skb_refdst;                                          
    void            (*destructor)(struct sk_buff *skb);                       
#ifdef CONFIG_XFRM                                                            
    struct  sec_path    *sp;                                                  
#endif                                                                        
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)       
    struct nf_conntrack *nfct;		//offset 84                                                
#endif                                                                        
#if IS_ENABLED(CONFIG_BRIDGE_NETFILTER)                                       
    struct nf_bridge_info   *nf_bridge;                                       
#endif
...//省略部分
};

static inline void nf_conntrack_get(struct nf_conntrack *nfct)
{
 if (nfct)
  atomic_inc(&nfct->use);	//出错位置，nfct为异常地址，访问use时出错
}

static inline void __nf_copy(struct sk_buff *dst, const struct sk_buff *src)
{
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
 dst->nfct = src->nfct;
 nf_conntrack_get(src->nfct);		//调用 nf_conntrack_get
 dst->nfctinfo = src->nfctinfo;
#endif
#if IS_ENABLED(CONFIG_BRIDGE_NETFILTER)
 dst->nf_bridge  = src->nf_bridge;
 nf_bridge_get(src->nf_bridge);
#endif
}
//__copy_skb_header函数
static void __copy_skb_header(struct sk_buff *new, const struct sk_buff *old) 
{                                                                             
    new->tstamp     = old->tstamp;                                            
    /* We do not copy old->sk */                                              
    new->dev        = old->dev;                                               
    memcpy(new->cb, old->cb, sizeof(old->cb));                                
    skb_dst_copy(new, old);                                                   
#ifdef CONFIG_XFRM                                                            
    new->sp         = secpath_get(old->sp);                                   
#endif                                                                        
    __nf_copy(new, old, false);		//在这里调用  __nf_copy 操作nfct相关                                          
                                                                              
...//省略部分                                                                    
                                                                              
}  
```

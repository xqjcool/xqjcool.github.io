---
title: "[Crash Analysis] System Crash Caused by Packet Decryption"
date: 2020-09-10
---

现网环境出现了crash，原因是"general protection fault: 0000 [#1]  "，也就是说访问了异常地址导致。

查看异常调用栈。

```bash
//...省略部分
 #6 [ffff88041fd43830] general_protection at ffffffff8168ef68
    [exception RIP: put_page+10]
    RIP: ffffffff8118edaa  RSP: ffff88041fd438e8  RFLAGS: 00010202
    RAX: 0000000000000040  RBX: 0000000000000002  RCX: 0000000000000010
    RDX: ffffea000a758960  RSI: ffff88041fd5a0a8  RDI: 2f302009302f3020
    RBP: ffff88041fd438f0   R8: ffffea000a758960   R9: ffff88041fb78980
    R10: 00000000000000cc  R11: 000000000000004e  R12: ffff8800775db4c0
    R13: ffff88020cf2c900  R14: ffff8800775db46a  R15: ffff8800775db44e
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #7 [ffff88041fd438f8] skb_release_data at ffffffff8155ea0f
 #8 [ffff88041fd43920] skb_release_all at ffffffff8155eae4
 #9 [ffff88041fd43938] consume_skb at ffffffff8155efdc
#10 [ffff88041fd43958] __dev_kfree_skb_any at ffffffff8156ee5d
#11 [ffff88041fd43980] TM_WanRx at ffffffffa03374d8 [testmod]
//...省略部分
```
直接原因是释放page时，page指针异常。异常值为 RDI：2f302009302f3020，把值作为page指针访问时触发异常地址访问保护机制。

```bash
crash> dis -r put_page+10
0xffffffff8118edb0 <put_page>:  data32 data32 data32 xchg %ax,%ax [FTRACE NOP]
0xffffffff8118edb5 <put_page+5>:        push   %rbp
0xffffffff8118edb6 <put_page+6>:        mov    %rsp,%rbp
0xffffffff8118edb9 <put_page+9>:        push   %rbx
0xffffffff8118edba <put_page+10>:       testq  $0xc000,(%rdi)    //访问page
```

在skb_release_data处理中将skb指针保存在寄存器$13中，且put_page并未对其改动，所以我们可以指导skb指针地址为 R13: ffff88020cf2c900。

```bash
crash> sk_buff ffff88020cf2c900
struct sk_buff {
//...
  _skb_refdst = 0, 
  sp = 0x0, 
  len = 76, 
  data_len = 0, 
  mac_len = 14, 
//...
  headers_start = 0xffff88020cf2c998, 
  skb_iif = 3, //...
  transport_header = 98, 
  network_header = 78, 
  mac_header = 78, 
//...
  headers_end = 0xffff88020cf2c9c8, 
//...
  tail = 190, 
  end = 192, 
  head = 0xffff8800775db400 "496\t 0/0/0/0\t 100,120,0,0,0,0,10,12\t 0,0,0,0,0,0,0\t 0,0,0,0,0,0, \220oBwE\376\356\070T\022<\b", 
  data = 0xffff8800775db472 "kV\235\201\215\245\003\064\341\266o\372\252c\n\005\325h\001\230\071\224Ͻ\275\276\276g\200\355d\215?\301\221q\347\316\316'\\\265\a`5\373\325\347\031C\246\320d\270\061}\364\f\220\036{\331Xx\021\322L(+R\327\372V\223\037\062e\265", <incomplete sequence \336>, 
  truesize = 768, 
//...
}
```

进而得到 skb_shared_info的内容 skb->head + skb->end = 0xffff8800775db400 + c0 = 0xffff8800775db4c0。

```bash
crash> skb_shared_info 0xffff8800775db4c0
struct skb_shared_info {
  nr_frags = 102 'f', 
  tx_flags = 222 '\336', 
  gso_size = 0, 
  gso_segs = 0, 
  gso_type = 0, 
  frag_list = 0x0, 
  hwtstamps = {
    hwtstamp = {
      tv64 = 0
    }, 
    syststamp = {
      tv64 = 0
    }
  }, 
  ip6_frag_id = 0, 
  dataref = {
    counter = 1
  }, 
  destructor_arg = 0x945424f52502009, 
  frags = {{
      page = {
        p = 0xffffea000a19cbc0
      }, 
      page_offset = 140, 
      size = 1350
    }, {
      page = {
        p = 0x2f302009302f3020	//异常page指针
      }, 
      page_offset = 807405872, 
      size = 808398895
    },
  }
//...
}
```

可以看到是在释放frags[1]的page时出现异常的。

skb->data_len = 0表示没有分片，所以skb_shared_info的nr_frags = 102 是异常值。

由于 skb_shared_info是紧跟在数据buffer的end之后，怀疑是修改数据buffer时异常将buffer之后的内容也修改了。

查看业务代码逻辑，我们在 TM_WanRx 处理中会对payload进行解密操作，使用的是aes对称解密，blocksize是16字节。

但是payload的长度是76字节，由于业务处理逻辑不严谨，导致会解密16 * 5 = 80字节，也就是tail之后多操作了4个字节。 

```c
//错误逻辑会导致将skb->tail之后的字节也当作密文解密了
    for (i = 0; i < skb->len; i += crypto_cipher_blocksize(tfm))
    {
        crypto_cipher_decrypt_one(tfm, dst + i, src + i);
    }
```

skb->tail=190 , skb->end = 192,也就是说会把end之后2个字节写坏。刚好是 nr_frags和tx_flags，然后在释放skb时，因为nr_frags不为0，依次释放page时，导致异常。

```bash
crash> rd 0xffff8800775db472 16		//查看payload内容
ffff8800775db472:  3403a58d819d566b 050a63aafa6fb6e1   kV.....4..o..c..
ffff8800775db482:  bdcf9439980168d5 8d64ed8067bebebd   .h..9......g..d.
ffff8800775db492:  27cecee77191c13f e7d5fb356007b55c   ?..q...'\..`5...
ffff8800775db4a2:  7d31b864d0a64319 7858d97b1e900cf4   .C..d.1}....{.Xx
ffff8800775db4b2:  fad7522b284cd211 de66b565321f9356   ..L(+R..V..2e.f.	//66是nr_frags de是tx_flags
ffff8800775db4c2:  0000000000000000 0000000000000000   ................
ffff8800775db4d2:  0000000000000000 0000000000000000   ................
crash> rd 0xffff8800775db4c0 -8                       //skb_shared_info开始
ffff8800775db4c0:  66                                 //nr_frags
crash> rd 0xffff8800775db4c1 -8
ffff8800775db4c1:  de                                 //tx_flags                .
```

触发问题的场景找到了，但是有个小问题。正常情况下对端加密后payload应该是16字节的整数倍，然后这边正常解密。什么情况导致payload长度错误呢？

查看系统log发现，在crash前有个配置变更，将报文传输方式由不加密修改为加密。配置是发向连接的两端设备，但是存在时间差，在这时收到了对端发来的不加密报文，被当作加密报文处理了，从而导致了该问题的发生。

---
title: "[Process Crash] Debugging and analysis of ifconfig process crash."
date: 2021-10-16
---

技术那边新遇到个问题，客户那里设备掉电，重启后发现设备无法联网。通过串口登上去发现除了环回口lo，其他所有接口都没有地址了。

尝试重启业务，重启设备还是不行。

尝试手动ifconfig配置ip，结果ifconfig进程崩溃了，提示segement fault。倒是ip address 可以正常查看接口。
![image](https://github.com/user-attachments/assets/952d6808-19f4-4339-a6cd-a32373e8c26e)
起初我怀疑是掉电导致网卡异常，导致配置丢失和ifconfig不能正常工作。于是查看日志信息和网卡信息。没有看到什么线索，于是让其也把ifconfig core文件发回来查看。
```c
[root@localhost core]# gdb /usr/sbin/ifconfig core-ifconfig-32676-1634254387
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-115.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /usr/sbin/ifconfig...Reading symbols from /usr/sbin/ifconfig...(no debugging symbols found)...done.
(no debugging symbols found)...done.
[New LWP 32676]

warning: .dynamic section for "/lib64/libc.so.6" is not at the expected address (wrong library or version mismatch?)

warning: .dynamic section for "/lib64/ld-linux-x86-64.so.2" is not at the expected address (wrong library or version mismatch?)

warning: Could not load shared library symbols for ڿÿ턙ÿÿȿÿ޿ÿ?ÿÿဨWÿÿhWÿÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿhWÿÿhWÿÿ瘿ÿ瘿ÿWÿÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿ瘿ÿחÿÿ.
Do you need "set solib-search-path" or "set sysroot"?
Core was generated by `ifconfig'.
Program terminated with signal 11, Segmentation fault.
#0  0x00007f8874c14c6f in __libc_csu_init ()
Missing separate debuginfos, use: debuginfo-install net-tools-2.0-0.22.20131004git.el7.x86_64
(gdb) bt
Python Exception <class 'gdb.MemoryError'> Cannot access memory at address 0x7f8874c0c09a: 
(gdb) 
```
连bt命令都无法展示，看起来不对劲。

那就看下crash地址吧

```c
(gdb) disass 0x00007f8874c14c6f
Dump of assembler code for function __libc_csu_init:
   0x00007f8874c14c20 <+0>:    push   %r15
   0x00007f8874c14c22 <+2>:    mov    %edi,%r15d
   0x00007f8874c14c25 <+5>:    push   %r14
   0x00007f8874c14c27 <+7>:    mov    %rsi,%r14
   0x00007f8874c14c2a <+10>:    push   %r13
   0x00007f8874c14c2c <+12>:    mov    %rdx,%r13
   0x00007f8874c14c2f <+15>:    push   %r12
   0x00007f8874c14c31 <+17>:    lea    0x203ef8(%rip),%r12        # 0x7f8874e18b30
   0x00007f8874c14c38 <+24>:    push   %rbp
   0x00007f8874c14c39 <+25>:    lea    0x203ef8(%rip),%rbp        # 0x7f8874e18b38
   0x00007f8874c14c40 <+32>:    push   %rbx
   0x00007f8874c14c41 <+33>:    sub    %r12,%rbp
   0x00007f8874c14c44 <+36>:    xor    %ebx,%ebx
   0x00007f8874c14c46 <+38>:    sar    $0x3,%rbp
   0x00007f8874c14c4a <+42>:    sub    $0x8,%rsp
   0x00007f8874c14c4e <+46>:    callq  0x7f8874c09ca0 <_init>
   0x00007f8874c14c53 <+51>:    test   %rbp,%rbp
   0x00007f8874c14c56 <+54>:    je     0x7f8874c14c76 <__libc_csu_init+86>
   0x00007f8874c14c58 <+56>:    nopl   0x0(%rax,%rax,1)
   0x00007f8874c14c60 <+64>:    mov    %r13,%rdx
   0x00007f8874c14c63 <+67>:    mov    %r14,%rsi
   0x00007f8874c14c66 <+70>:    mov    %r15d,%edi
   0x00007f8874c14c69 <+73>:    callq  *(%r12,%rbx,8)
   0x00007f8874c14c6d <+77>:    add    $0x1,%rbx        //0x00007f8874c14c6f ？
   0x00007f8874c14c71 <+81>:    cmp    %rbp,%rbx        //0x00007f8874c14c6f ？
   0x00007f8874c14c74 <+84>:    jne    0x7f8874c14c60 <__libc_csu_init+64>
   0x00007f8874c14c76 <+86>:    add    $0x8,%rsp
   0x00007f8874c14c7a <+90>:    pop    %rbx
   0x00007f8874c14c7b <+91>:    pop    %rbp
   0x00007f8874c14c7c <+92>:    pop    %r12
   0x00007f8874c14c7e <+94>:    pop    %r13
   0x00007f8874c14c80 <+96>:    pop    %r14
   0x00007f8874c14c82 <+98>:    pop    %r15
   0x00007f8874c14c84 <+100>:    retq   
End of assembler dump.
```
从反汇编上看， 0x00007f8874c14c6f 这个指令地址不太对， 它在 0x00007f8874c14c6d和 0x00007f8874c14c71 中间。除此之外没有别的有用信息了，思来想去好久没明白为什么会这样。

        过了一会儿，脑中突然出现个念头，难道ifconfig程序被破坏了？

于是对设备的ifconfig进行MD5查看
![image](https://github.com/user-attachments/assets/f8111246-9dca-4375-80ca-a5ebdc574ed3)
 对比正常设备的ifconfig MD5
 ![image](https://github.com/user-attachments/assets/786ef9b7-0036-4e4a-8e27-0284197b0dd1)

 果然MD5值不一样。

于是赶紧传输正常的ifconfig程序导致问题设备上，手动执行后，果然没有crash，bingo！~一切又都恢复正常了。

![image](https://github.com/user-attachments/assets/3ff18b1a-223f-46cb-bdbe-dac2269f8abf)

如此看来是异常掉电，导致ifconfig程序被破坏(有点匪夷所思)，导致ifconfig执行后segment fault，影响到公司业务相关脚本(这些脚本会通过ifconfig配置ip)，导致无法联网。



---
title: "[Crash Analysis] System crash caused by array out-of-bounds access"
date: 2020-06-23
---

公司设备在项目中遇到了crash问题，查看coredump，是最长前缀匹配的模块中处理异常导致。

现象1：查询异常

```bash
"general protection fault: 0000 [#1] SMP "

    [exception RIP: _test_ipset_seg_lpm_query+93]
    RIP: ffffffffa058ce9d  RSP: ffff88016951f578  RFLAGS: 00010207
    RAX: 80000010241c2865  RBX: ffff88102b96c008  RCX: 0000000000000008
    RDX: ffff88016951f6fc  RSI: 80000010241c2be9  RDI: 000000000000004b
    RBP: ffff88016951f598   R8: ffffffffa058ce40   R9: ffffffffa057fc7d
    R10: ffff88102d319a60  R11: ffffea004073fb80  R12: ffff88016951f6fc
    R13: 000000000a174bc0  R14: 0000000000000000  R15: ffff88102c2d2668
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
crash> dis _test_ipset_seg_lpm_query+93
0xffffffffa058ce9d <_test_ipset_seg_lpm_query+93>:      cmpb   $0x0,(%rsi) 
```
经分析是seg指针地址异常，seg->seg_array_next  操作导致系统crash。这是query查询操作，起初以为是并发处理保护没有做好，seg指针在其他地方被释放导致。
查阅代码后，发现add/del/query 有对应的锁保护，没有发现错误。


随后出现问题现象2：提示地址访问异常。

```bash
BUG: unable to handle kernel paging request at 0000000000497ff0
    [exception RIP: _test_ipset_seg_lpm_del_rule+175]
    RIP: ffffffffa053801f  RSP: ffff880feb6bbc08  RFLAGS: 00010206
    RAX: ffff88102800c008  RBX: ffff881026f09808  RCX: 0000000000000000
    RDX: 00000000000000c8  RSI: 000000000ac80100  RDI: ffff881026f0980c
    RBP: ffff880feb6bbc50   R8: 00000000000ac801   R9: 00000000004973f0
    R10: 000000001f308101  R11: ffffea00407cc200  R12: ffff881028bdb008
    R13: 000000000000001d  R14: ffff881027411008  R15: 000000000ac80100
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 
crash> dis _test_ipset_seg_lpm_del_rule+175
0xffffffffa053801f <_test_ipset_seg_lpm_del_rule+175>:   mov    0xc00(%r9),%rax

```
经分析是指针数组seg_array_next[200] = 中指针地址错误, 导致异常地址访问，系统crash。
这个是delete操作，怀疑方向转到了重复释放上，然而查阅代码，没找重复释放问题。

增加add/del相关log，并对内存的申请和释放增加了wrapper函数，打印对应地址。
查看相关log并没有发现问题地址的释放。

思路一度陷入僵局，随后增加了更多的log，crash后，查看log文件，发现一个诡异问题，在某个时刻，seg->seg_array_next指针发生了改变，并且只是第三个字节被置0。如

```bash
 //为了好理解 小端字节序 从小到大排列
 08 40 93 a8 00 88 ff ff 被改成
 08 40 00 a8 00 88 ff ff
```
这个很明显是地址被篡改，回过头看seg的格式，发现如果Ele数组越界，depth正好对应seg_array_next指针的第三个字节。于是推断是update Ele操作时，数组index为256，修改depth=0 导致的问题。

```bash
typedef struct _TEST_SET_LPM_DIR
{
    struct {
        u8  ref;
        u8  seg;
        u8  depth;	//注意这里
        u32 ipaddr;
        u32 data;
    }   Ele[256];
    TEST_SET_LPM_DIR   **seg_array_next;
} TEST_SET_LPM_DIR; 
```
有了方向后，增加对应的log，重现问题。发现确实存在index=256的情况。仔细审视代码发现是一个手误，本来应该是index= var + i；结果写成了 index = var + 1；在代码中存在var=255的可能，于是导致了数组越界。

```bash
[  324.149807] [_test_ipset_seg_lpm_create:199] Seg:ffff880035820008 Seg->seg_array_next:ffff880035826008
...//省略无关
[  325.668351] [_test_ipset_seg_lpm_update:484] Seg:ffff880035820008 var:255 Seg->seg_array_next:ffff880035006008

```

修复后，验证通过，问题解决。

但这个问题确实比较隐蔽，只是修改了对应地址的一个字节，导致初期没有怀疑到数组越界，浪费了不少时间。

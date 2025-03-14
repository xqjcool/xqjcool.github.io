---
title: "[KASAN] Stack-out-of-bounds issue analysis and debugging."
date: 2024-08-27
---

# 起因

我们最近在定位一个棘手的内存问题， 常规手段没能奏效，于是出了个开启KASAN的debug image，运行在测试环境，希望能帮助定位问题。
运行后帮我们发现了几个存在很久的代码问题。

# 问题现象
首先时发现一例栈越界的问题。log如下：

```bash
Aug 24 02:21:56 Test-Dev kern.err kernel: [85593.844504] ==================================================================
Aug 24 02:21:56 Test-Dev kern.err kernel: [85593.930986] BUG: KASAN: stack-out-of-bounds in nis_compare+0x2db/0x360
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.004927] Read of size 4 at addr ffff8882f323f510 by task balanced/27501
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.083025]
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100815] CPU: 2 PID: 27501 Comm: balanced Tainted: G           O    4.14 #12
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100816] Hardware name: GIGABYTE MN32-EC1-F5/MN32-EC1-F5, BIOS F03 06/04/2019
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100817] Call Trace:
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100820]  dump_stack+0x57/0x6d
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100823]  print_address_description+0x74/0x238
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100825]  ? nis_compare+0x2db/0x360
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100827]  kasan_report_error.cold+0x8a/0x1cd
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100829]  __asan_report_load4_noabort+0x70/0x80
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100831]  ? nis_compare+0x2db/0x360
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100832]  ? nis_find_via_gw+0x180/0x180
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100833]  nis_compare+0x2db/0x360
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100835]  ? nis_find_via_gw+0x180/0x180
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100837]  foreach_net_only+0xab/0x100
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100838]  filter_foreachate_cleanup+0x179/0x270
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100840]  ? filter_alloc_hashtable+0x130/0x130
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100841]  ? nis_find_via_gw+0x180/0x180
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100843]  filter_iterate_cleanup_net+0xb6/0x110
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100844]  ? clear_all_conntrack_timeout+0xd0/0xd0
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100845]  ? nis_find_via_gw+0x180/0x180
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100848]  ? rtnl_notify+0x8e/0xe0
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100849]  ? rtmsg_fib+0x232/0x490
Aug 24 02:21:56 Test-Dev kern.warn kernel: [85594.100851]  nis_tbl_remove_conn_handle+0x88/0x90
//中间部分省略
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.118799] The buggy address belongs to the page:
Aug 24 02:21:56 Test-Dev kern.emerg kernel: [85594.176106] page:ffffea000bcc8fc0 count:0 mapcount:0 mapping:          (null) index:0x0
Aug 24 02:21:56 Test-Dev kern.emerg kernel: [85594.271890] flags: 0x8000000000000000()
Aug 24 02:21:56 Test-Dev kern.alert kernel: [85594.317754] raw: 8000000000000000 0000000000000000 0000000000000000 00000000ffffffff
Aug 24 02:21:56 Test-Dev kern.alert kernel: [85594.410420] raw: 0000000000000000 0000000100000001 ffff88842d003180 0000000000000000
Aug 24 02:21:56 Test-Dev kern.alert kernel: [85594.503083] page dumped because: kasan: bad access detected
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.569745]
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.587533] Memory state around the buggy address:
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.644836]  ffff8882f323f400: 00 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1 00
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.731259]  ffff8882f323f480: 00 00 f3 f3 f3 f3 f3 00 00 00 00 00 00 f1 f1 f1
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.817680] >ffff8882f323f500: f1 04 f3 f3 f3 00 00 00 00 00 00 00 00 00 00 00
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.904107]                          ^
Aug 24 02:21:56 Test-Dev kern.err kernel: [85594.948932]  ffff8882f323f580: 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1 f1 f1
Aug 24 02:21:56 Test-Dev kern.err kernel: [85595.035352]  ffff8882f323f600: 00 00 00 00 00 f3 f3 f3 f3 f3 00 00 00 00 00 00
Aug 24 02:21:56 Test-Dev kern.err kernel: [85595.121776] ==================================================================
```

# 问题分析

首先我们了解到这是一个栈越界问题，表明访问栈上的数据时越界了。触发时的函数是nis_compare。
但这不是说nis_compare函数的栈越界了，也有可能是其他函数的栈。

需要仔细检查 nis_compare都访问了那些变量，那些可能存在越界访问。通过反汇编分析，目标锁定在入参 arg上。

```c
struct nis_compare_data {
        int table;
        int outif;
        unsigned int gateway;
};

static int nis_compare(struct nf_conn* ct, void* arg)
{       
        nf_conn_nis *nis = NULL; 
        struct nis_compare_data* data = (struct nis_compare_data*)arg;
//部分省略 
        if (data->gateway == nis->gataway)
//部分省略
        
        return 0;
}

```
会不会这个arg是栈上的结构，然后在这里转为 nis_compare_data 导致访问越界？

# 追溯 arg
向前追溯一直到nis_tbl_remove_conn_handle函数， 这个函数将一个local整型的变量table的地址传给函数filter_iterate_cleanup_net，最终作为arg传给nis_compare。
明显这会导致访问data->gateway时越界。

PS:这里对新手有点迷惑性，入参table不是上一个函数传过来的么，怎么会在栈上？由于需要使用table地址作为参数，系统会把table存在栈上，再把栈地址传给filter_iterate_cleanup_net。

```c
int nis_tbl_remove_conn_handle(struct net* net, int table)
{
//部分省略
                filter_iterate_cleanup_net(net, nis_compare, (void*)&table, 0, 0);
//部分省略
```

# 查看反汇编
从返回便上，我们可以看到开启kasan后，会在栈上占用更多空间，同时也在shadow region中对栈变量区域进行标记。

0xf1表示左边界，0xf3表示右边界。栈区内存大致如下：
可以看到kasan用左右边界将局部变量保护起来了。一旦访问到0xf1或0xf3，即视为越界。04代便这8个byte只有4个byte可读，因为table是整型4个字节。
如果把tableparam的地址当作 nis_compare_data去访问，那么我们访问结构中的gateway，因为对应shadow region这里是0xf3，因此触发kasan告警。

```bash
shadown    	    stack mem
region
			|       rbp        |
			|       rbx        |
|f3|		|                  |
|f3|		|                  |
|f3|		|                  |	|gatway	|
|04|		|  table param     |	|table	|outif	|
|f1|		|                  |
|f1|		|0xffffffff81b99b10|
|f1|		|0xffffffff8298022e|
|f1|		|0x41b58ab3        |
```

```c
ffffffff81b99b10 <nis_tbl_remove_conn_handle>:
ffffffff81b99b10:       e8 8b 7c 46 00          callq  ffffffff820017a0 <__fentry__>
ffffffff81b99b15:       48 b8 00 00 00 00 00    movabs $0xdffffc0000000000,%rax
ffffffff81b99b1c:       fc ff df
ffffffff81b99b1f:       55                      push   %rbp
ffffffff81b99b20:       48 89 e5                mov    %rsp,%rbp
ffffffff81b99b23:       53                      push   %rbx
ffffffff81b99b24:       48 8d 5d b8             lea    -0x48(%rbp),%rbx
ffffffff81b99b28:       48 c1 eb 03             shr    $0x3,%rbx
ffffffff81b99b2c:       48 01 d8                add    %rbx,%rax
ffffffff81b99b2f:       48 83 ec 40             sub    $0x40,%rsp
ffffffff81b99b33:       48 c7 45 b8 b3 8a b5    movq   $0x41b58ab3,-0x48(%rbp)
ffffffff81b99b3a:       41
ffffffff81b99b3b:       48 c7 45 c0 2e 02 98    movq   $0xffffffff8298022e,-0x40(%rbp)
ffffffff81b99b42:       82
ffffffff81b99b43:       48 c7 45 c8 10 9b b9    movq   $0xffffffff81b99b10,-0x38(%rbp)
ffffffff81b99b4a:       81
ffffffff81b99b4b:       c7 00 f1 f1 f1 f1       movl   $0xf1f1f1f1,(%rax)		//kasan magic
ffffffff81b99b51:       c7 40 04 04 f3 f3 f3    movl   $0xf3f3f304,0x4(%rax)	//kasan magic
ffffffff81b99b58:       89 75 d8                mov    %esi,-0x28(%rbp)			//table param

```


# 对比验证告警中的buggy address
"Read of size 4 at addr ffff8882f323f510 by task balanced/27501", 意思是在ffff8882f323f510地址上读取4个byte，也就是一个整型的值。
让我们再次看一下buggy address。可以看到指示的位置刚好是04上面的f3。

# 后记
KASAN确实是个好工具，但是开启后性能消耗太大。
所以在测试环境可以使用，生产环境不能使用。

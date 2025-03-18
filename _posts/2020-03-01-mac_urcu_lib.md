---
title: "[macOS] Crash Analysis: Program Crash Caused by Calling urcu Library Functions"
date: 2020-03-01
---

产品运行在mac os上，使用了userspace-rcu库来提高性能。但是在运行中总是会出现crash，crash位置是urcu函数调用内部。
userspace-rcu要求每个会使用urcu API的线程都要在开始时调用 register urcu thread API，结束时调用 unregister urcu thread。大多数异常情况是没有按照这个要求编码导致的。

## 1. 检查是否有线程未调用reg/unreg urcu thread API
因为程序运行时线程过多(超过30个)，且一个一个手动检查代码容易遗漏。于是下载了userspace-rcu的源码，经研究发现其支持debug功能，开启debug后，可以帮助我们检查是否有线程未调用reg urcu thread API。
### 1.1 编译带debug功能能的urcu库
进入userspace-rcu目录，执行./configure 添加 "--enable-rcu-debug"选项来开启urcu的debug功能。

```powershell
./configure --host x86_64-apple-darwin19 CFLAGS="-fPIC" --disable-shared --enable-rcu-debug
```
然后编译出liburcu库文件。
### 1.2 重新编译可执行程序并验证问题
因为使用了带debug功能的urcu库，只要没有reg urcu thread，在调用rcu_read_lock API时会触发assert异常，很快暴露了一个存在问题的地方。
修改后重新验证，没有再触发assert异常了，表明用到urcu API的线程，都调用了reg urcu thread。

但是运行一段时间后，还是会有crash，crash位置是synchronize_rcu函数中。看来还需要进一步分析。
## 2. 分析异常调用栈
xcode调试运行程序，异常时调用栈如下：
![image](https://github.com/user-attachments/assets/d8664228-212f-4f3d-9f88-877fa15ef32e)

完整栈层次如下：
![image](https://github.com/user-attachments/assets/74a82ae4-f34c-47a5-abcc-ce1755946e1d)

应用调用synchronize_rcu后，最终在_cds_list_del时crash。
对照源码和反汇编可知，直接错误原因是next为NULL，在执行next->prev = prev;时触发异常地址访问。
也就是为何错误提示是addr=0x8 (next->prev相对于next偏移是0x8)。

```c
static inline
void __cds_list_del(struct cds_list_head *prev, struct cds_list_head *next)
{
 next->prev = prev;
 prev->next = next;
}
```

阅读urcu源码可知，wait_for_readers遍历全局registry链表，调用cds_list_move将特定的node从链表移走。怀疑链表上某个node已经坏掉，导致问题发生。于是通过lldb， p &registry获取到了全局链表头，然后 p *(cds_list_head *)addr依次遍历查看，好在链表不长，很快找到问题node。该node的next指针确实是NULL。
next指针是怎么变成NULL的呢？起初怀疑是并发操作改坏的，但是查看代码，链表操作有mutex锁保护，所以不是并发导致。

为了进一步分析，我在系统初始化完毕时设置了个断点，进入断点后，遍历registry链表，记录了每个node的地址和node中保存的线程tid。并对当前运行的个线程pid做对比，建立映射。

```powershell
这里注意：xcode中显示的线程id和我们所说的通过pthread_self()打印出来的tid是不一样的。想要获取线程tid，需要在lldb中切换到对应线程，再执行pthread_self()调用获取。例如：
thread select 11	//切换到线程11
call (unsigned long)pthrea_self()	//调用api获取当前线程tid，需要强转为unsigned long
```

然后继续运行，在进程crash后，再遍历registry链表，找到问题node的地址。和刚开始记录的链表对比，发现问题节点是初始时就注册的一个node，并且node的prev和next都是好的。再看node对应的线程，是程序初始化操作所在线程。经过查看程序初始化操作，终于找到了问题原因。

我们程序在另一个平台初始化操作是在主线程执行的，初始化时调用register urcu thread，然后整个运行中不会退出，在程序退出时，调用unregister urcu thread。
移植到mac os平台后，初始化操作是单独一个线程，执行完初始化操作后就退出了。而初始化时 register到 urcu 全局链表的node，没有调用unregster释放。导致这块内存释放了，但是node还在urcu链表上。node结构内容变得不可预测，最终导致问题发生。

修改为在初始化线程结束的时候，添加unregister urcu thread操作。调试验证OK。

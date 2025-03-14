---
title: "[Module Unloading Failure] Troubleshooting the Failure of Business Module Unloading."
date: 2021-04-23
---


设备重启服务会先卸载业务模块，然后再重新加载。线上有台设备在重启业务时卡住。

## 查看服务状态
```bash
[root@localhost ~]# systemctl status BSN
● BSN.service - APXBSN BSN SERVICE
   Loaded: loaded (/usr/lib/systemd/system/BSN.service; enabled; vendor preset: disabled)
   Active: active (exited) since Fri 2021-04-16 13:36:17 SAST; 32min ago
  Process: 16376 ExecStart=/bin/apxbsn/bsnctl start (code=exited, status=0/SUCCESS)
 Main PID: 16376 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/BSN.service
           └─control
             └─31969 rmmod bsnmod
```

提示是卸载bsnmod 业务模块卡住了。

## 查看卸载进程状态

```bash
[root@localhost ~]# ps aux|grep rmmod
root     22551  0.0  0.0 112712   964 pts/5    S+   14:08   0:00 grep --color=auto rmmod
root     31969  0.0  0.0  13112   800 ?        D    Feb25   0:00 rmmod bsnmod
```

进程状态D(TASK_UNINTERRUPTIBLE)，不可中断的睡眠状态。说明在等待某种条件唤醒。

## 查看进程栈信息

```bash
[root@localhost ~]# cat /proc/31969/stack
[<ffffffff810a6fec>] flush_workqueue+0x12c/0x5b0
[<ffffffff810a7485>] flush_scheduled_work+0x15/0x20
[<ffffffffa03d8399>] apxbsnIoRelease+0x29/0x160 [bsnmod]
[<ffffffffa03d9753>] _BSN_BaseDestroy+0x43/0x1a0 [bsnmod]
[<ffffffffa03d99d5>] BSN_BaseDestroy+0x65/0xa0 [bsnmod]
[<ffffffffa03cac9f>] BSN_Release+0xef/0x170 [bsnmod]
[<ffffffffa042d7ca>] _BSN_WanRelease+0x3a/0x70 [bsnmod]
[<ffffffffa0430480>] BSN_WanDel+0x100/0x260 [bsnmod]
[<ffffffffa0432440>] BSN_NetWanRelease+0x50/0x60 [bsnmod]
[<ffffffffa042d60f>] BSN_NetRelease+0x4f/0x110 [bsnmod]
[<ffffffffa03959db>] cleanup_module+0x1b/0xd0 [bsnmod]
[<ffffffff810fe3db>] SyS_delete_module+0x16b/0x2d0
[<ffffffff81697809>] system_call_fastpath+0x16/0x1b
[<ffffffffffffffff>] 0xffffffffffffffff
```

可以看到卡在了 flush_workqueue 这里。
 业务模块会调用 flush_scheduled_work 清理工作队列，保证工作队列中没有业务模块的work存在。
 

```c
void flush_workqueue(struct workqueue_struct *wq)
{
//...
	//flush_workqueue 卡在这里
    wait_for_completion(&this_flusher.done);

//...
}
```

这是等待工作队列flush完成后的唤醒信号。卡在这里说明有工作队列尚未完成。

## 查看卡住的工作队列

```bash
[root@localhost ~]# ps aux|grep D
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      1410  0.0  0.0      0     0 ?        S    Jan12  20:29 [BSN_DataConf]
root      4790  0.0  0.0      0     0 ?        D    Feb25   0:01 [kworker/1:2]	//卡住
root      7297  0.0  0.2 112920  4360 ?        Ss   Apr16   0:25 /usr/sbin/sshd -D
root     29794  0.0  0.0 112716   944 pts/4    S+   04:05   0:00 grep --color=auto D
root     31969  0.0  0.0  13112   800 ?        D    Feb25   0:00 rmmod bsnmod
```

可以看到工作队列 [kworker/1:2]处于 D(TASK_UNINTERRUPTIBLE)，不可中断的睡眠状态。该工作队列的长时间卡住，导致了 flush_workqueue 迟迟得不到完成信号的原因。此时我们已经知道了 bsnmod卸载卡住问题的原因。 那么 [kworker/1:2]为何卡在这里呢？

## 查看异常工作队列进程栈

```bash
[root@localhost ~]# cat /proc/4790/stack
[<ffffffffa030ef25>] ttm_eu_reserve_buffers+0x205/0x320 [ttm]
[<ffffffffa0328961>] qxl_release_reserve_list+0x51/0x110 [qxl]
[<ffffffffa0326686>] qxl_draw_opaque_fb+0xe6/0x3c0 [qxl]
[<ffffffffa0322f92>] qxl_fb_dirty_flush+0x1a2/0x260 [qxl]
[<ffffffffa0323069>] qxl_fb_work+0x19/0x20 [qxl]
[<ffffffff810a845b>] process_one_work+0x17b/0x470
[<ffffffff810a9296>] worker_thread+0x126/0x410
[<ffffffff810b0a4f>] kthread+0xcf/0xe0
[<ffffffff81697758>] ret_from_fork+0x58/0x90
[<ffffffffffffffff>] 0xffffffffffffffff
```
查看工作队列进程栈，发现卡在了 ttm_eu_reserve_buffers处理。这是ttm模块的业务函数。这里的ttm和qxl都是内核gpu相关驱动。

对应相关代码块如下：

```c
static __always_inline int __sched
__mutex_lock_common(struct mutex *lock, long state, unsigned int subclass,
		    struct lockdep_map *nest_lock, unsigned long ip,
		    struct ww_acquire_ctx *ww_ctx)
{
//... 省略
	spin_lock_mutex(&lock->wait_lock, flags);

	/* once more, can we acquire the lock? */
	if (MUTEX_SHOW_NO_WAITER(lock) && (atomic_xchg(&lock->count, 0) == 1))
		goto skip_wait;

	debug_mutex_lock_common(lock, &waiter);
	debug_mutex_add_waiter(lock, &waiter, task_thread_info(task));

	/* add waiting tasks to the end of the waitqueue (FIFO): */
	list_add_tail(&waiter.list, &lock->wait_list);
	waiter.task = task;

	lock_contended(&lock->dep_map, ip);
	for (;;) {
		/*
		 * Lets try to take the lock again - this is needed even if
		 * we get here for the first time (shortly after failing to
		 * acquire the lock), to make sure that we get a wakeup once
		 * it's unlocked. Later on, if we sleep, this is the
		 * operation that gives us the lock. We xchg it to -1, so
		 * that when we release the lock, we properly wake up the
		 * other waiters:
		 */
		if (MUTEX_SHOW_NO_WAITER(lock) &&
		    (atomic_xchg(&lock->count, -1) == 1))
			break;

		/*
		 * got a signal? (This code gets eliminated in the
		 * TASK_UNINTERRUPTIBLE case.)
		 */
		if (unlikely(signal_pending_state(state, task))) {
			ret = -EINTR;
			goto err;
		}

		if (!__builtin_constant_p(ww_ctx == NULL) && ww_ctx->acquired > 0) {
			ret = __mutex_lock_check_stamp(lock, ww_ctx);
			if (ret)
				goto err;
		}

		__set_task_state(task, state);

		/* didn't get the lock, go to sleep: */
		spin_unlock_mutex(&lock->wait_lock, flags);
		schedule_preempt_disabled();	//因为没有得到锁，调用schedule将自己调度出去，等待下次检查
		spin_lock_mutex(&lock->wait_lock, flags);
	}
//... 省略
}
```

从代码逻辑可知，获取lock失败后，将进程加入到 lock的等待链表。随后循环中进行 判断lock是否满足条件，schedule调度，待重新获得调度后，再次判断。

从栈信息可知，进程将自己设为 TASK_UNINTERRUPTIBLE，schedule调度出去后。sock条件迟迟没能满足，所以没有重新唤醒，导致进程一直处于 D(TASK_UNINTERRUPTIBLE) 状态。

问题到这里基本告一段落。本次问题不是业务模块代码造成的。看起来像是内核gpu的ttm/qxl相关代码导致的。

出于好奇心，又尝试进一步定位ttm 锁不释放的原因。

## 强制产生core文件并使用crash工具打开
和相关人员沟通并得到肯定答复后。我执行 echo c > /proc/sysfs-trigger，强制产生了core文件。随后将core文件拷贝下来，使用crash工具打开。

```bash
crash> ps | grep kworker/1:2
   4790      2   1  ffff88003570de20  UN   0.0       0      0  [kworker/1:2]	//找到进程
crash> task_struct.stack ffff88003570de20		//找到栈信息
  stack = 0xffff8800506d0000
crash> search -s 0xffff8800506d0000 ffffffffa030ef25	//找到 ttm_eu_reserve_buffers+517所在位置
ffff8800506d3bd8: ffffffffa030ef25 
crash> rd -s ffff8800506d3b00 64	//查看栈空间
ffff8800506d3b00:  ffff880035acf1e0 ffff88003570de20 
ffff8800506d3b10:  ffff880035acf1e4 ffff8800798bd888 
ffff8800506d3b20:  00000000ffffffff ffff8800506d3b38 
ffff8800506d3b30:  schedule_preempt_disabled+41 ffff8800506d3ba0 
ffff8800506d3b40:  __ww_mutex_lock_slowpath+423 ffff880035acf1e8 
ffff8800506d3b50:  ffff880035acf1e8 ffff880035acf1e8 
ffff8800506d3b60:  ffff88003570de20 000000001c97ce4b 
ffff8800506d3b70:  qxl_release_list_add+92 ffff880035acf1e0 
ffff8800506d3b80:  ffff8800798bd888 ffff8800798bd8a0 
ffff8800506d3b90:  ffff880035acf058 ffff88005491de80 
ffff8800506d3ba0:  ffff8800506d3bd0 __ww_mutex_lock+95 
ffff8800506d3bb0:  ffff88005491de80 ffff8800798bd8a0 
ffff8800506d3bc0:  ffff8800798bd8a0 ffff880035acf058 
ffff8800506d3bd0:  ffff8800506d3c28 ttm_eu_reserve_buffers+517 
ffff8800506d3be0:  ffff88003659dec0 0000000000000000 
ffff8800506d3bf0:  000000001c97ce4b ffff8800798bd888 
ffff8800506d3c00:  ffff880036668000 ffff8800798bd800 
ffff8800506d3c10:  ffff8800798bd8a0 ffff8800798bd888 
ffff8800506d3c20:  ffff8800506d3d38 ffff8800506d3c60 
ffff8800506d3c30:  qxl_release_reserve_list+81 ffff880036668000 
ffff8800506d3c40:  0000000000000020 0000000000000008 
ffff8800506d3c50:  0000000000001000 ffff8800506d3d38 
ffff8800506d3c60:  ffff8800506d3d08 qxl_draw_opaque_fb+230 
ffff8800506d3c70:  000000001c97ce4b ffff880036668000 
ffff8800506d3c80:  0000000000000000 0000000000000120 
ffff8800506d3c90:  ffffc90000c21000 0000012000000008 
ffff8800506d3ca0:  0000001000000000 ffff8800798bd800 
ffff8800506d3cb0:  ffff88005491d480 0000000000000000 
ffff8800506d3cc0:  ffff8800506d3cd0 000000001c97ce4b 
ffff8800506d3cd0:  ffff88007a648590 000000001c97ce4b 
ffff8800506d3ce0:  ffff8800364f1800 0000000000000000 
ffff8800506d3cf0:  0000000000000120 0000000000000010 
```

通过进程，找到栈底，再找到问题位置。然后将栈信息打出来。
这时我们获取到比 /proc/4790/stack 更详细的信息。

## 研究lock信息
我们主要关注 __ww_mutex_lock_slowpath 的处理。从栈中找到 lock的地址为 ffff880035acf1e0。
查看lock内容如下：

```bash
crash> struct mutex ffff880035acf1e0
struct mutex {
  count = {
    counter = -1	//锁
  }, 
  wait_lock = {
    {
      rlock = {
        raw_lock = {
          {
            head_tail = 131074, 
            tickets = {
              head = 2, 
              tail = 2
            }
          }
        }
      }
    }
  }, 
  wait_list = {
    next = 0xffff8800506d3b50, 		//waiter
    prev = 0xffff8800506d3b50
  }, 
  owner = 0xffff88003570de20, 		//owner
  {
    osq = 0x0, 
    __UNIQUE_ID_rh_kabi_hide2 = {
      spin_mlock = 0x0
    }, 
    {<No data fields>}
  }
}
crash> mutex_waiter 0xffff8800506d3b50	//查看waiter 
struct mutex_waiter {
  list = {
    next = 0xffff880035acf1e8, 
    prev = 0xffff880035acf1e8
  }, 
  task = 0xffff88003570de20
}

```

可以看到 __mutex_lock_slowpath在调用 schedule_preempt_disabled之前，已经将 waiter 加到 lock的 wait_list  链表。随后进入休眠，等待lock释放后将其唤醒。

不知道读者有没有注意到，lock的owner也是 0xffff88003570de20，即当前 [kworker/1:2] 进程！

owner表示当前锁的持有者。这表示[kworker/1:2] 进程已经持有了 lock，然后在 __mutex_lock_slowpath 函数中，再次尝试持有lock失败，然后将自己挂在waiter 链表后，进入休眠，等待唤醒。这导致了mutex死锁发生。

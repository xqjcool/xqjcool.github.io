---
title: "Issues with `pthread_cond_signal` / `pthread_cond_timedwait` Usage"
date: 2021-04-08
---

## 问题现象
项目上遇到一个问题，报文交互经常延时过高，客户非常不满。

## 初步抓包分析
通过抓包发现问题出在从物理口收包到我们模块处理时。也就是说不是网络中异常导致的延时过高。是我们处理过程中产生的。但是但从抓包很难确认延时产生在什么位置。需要进一步确认。
 
 
## 报文打点确认
于是对报文进行了处理，使其记录timestamp，然后在各个关键位置判断当前时间和报文timestamp的差值，如果超过一定范围，则通过log打印出来。
通过打点，确认问题发生在 业务模块将 报文放入队列，到主线程从队列取包处理时。

这部分的简单逻辑如下：
 - 模块从物理口收到报文后，会将其放在一个队列中，然后通过 pthread_cond_signal 通知主线程进行处理。
 - 主线程loop通过pthread_cond_timedwait等待信号或者超时。如果是信号唤醒，则从队列中取包进行处理。如果是超时，则处理一些异步事件。处理完毕后继续通过 pthread_cond_timedwait 等待下一次。

曾怀疑过是不是 pthread_cond_timedwait 唤醒太慢导致，增加debug log发现不是。那为什么报文到达队列后，等待很久才被主线程取出处理呢？

## 恍然大悟
回头又审视 pthread_cond_signal 函数，manual上写：

```bash
The pthread_cond_signal() function shall unblock at least one of the threads that are blocked on the specified condition variable cond (if any  threads  are  blocked  on cond).
```

pthread_cond_signal唤醒在制定cond条件上被阻塞的线程。如果没有线程被block ，那么pthread_cond_signal就相当于空操作了。
于是明白了为什么入队后的报文没有及时被处理：

 - 报文入队后，pthread_cond_signal去唤醒主线程。与此同时主线程的pthread_cond_timedwait 已经**超时**唤醒。pthread_cond_signal信号无效忽略。
 - 因为主线程pthread_cond_timedwait是超时唤醒，并未进行队列取包处理，只是处理异步事件。处理完毕后调用pthread_cond_timedwait等待下一次。
 - 直到某个报文入队，pthread_cond_signal去唤醒主线程，且主线程正在pthread_cond_timedwait休眠等待尚未唤醒和超时。此时主线程被signal信号唤醒，从队列中取包处理。此时之前队列中积压的报文都得到了处理，但是较早入队的报文处理时间间隔就会很高。
 
 ## 修复操作
 知道原因后，修复起来就简单了。之前主线程对信号唤醒还是超时唤醒进行了区分操作（超时唤醒假定了队列中没有包问，不处理报文队列）。实际上不需要，无论哪种唤醒都对队列进行取包处理。这样就能避免该问题的发生。

 ## 后记
 之前只关注了 pthread_cond_signal 的惊群问题，没注意到pthread_cond_timedwait唤醒间隙，pthread_cond_signal无效问题 。


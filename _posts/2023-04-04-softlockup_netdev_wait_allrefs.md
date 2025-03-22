---
title: "[crash analysis]Crash caused by failure to release device with a non-zero index"
date: 2023-04-04
---
# 设备索引不为0无法释放导致的crash

## 问题现象
客户上报设备联网后不停的reboot，断网后取得debug信息，发现有coredump文件。

## 初步分析

从core log上可以看到：
```
[  414.230967] unregister_netdevice: waiting for vlan2772 to become free. Usage count = 1
[  414.325702] rt_cache_flush
[  414.325702] dev vlan2772 unregister flush rt
[  415.350967] rt_cache_flush
[  415.350968] dev vlan2772 unregister flush rt
[  416.375967] rt_cache_flush
[  416.375968] dev vlan2772 unregister flush rt
[  417.398968] rt_cache_flush
[  417.398969] dev vlan2772 unregister flush rt
[  418.422966] rt_cache_flush
[  418.422968] dev vlan2772 unregister flush rt
[  418.678966] BUG: netdev_wait_allrefs: refcnt not release and stuck in dead loop!
[  418.767422] Kernel panic - not syncing: softlockup: netdev_wait_allrefs hung tasks
```
根据提示找到对应代码：

```c
static void netdev_wait_allrefs(struct net_device *dev)
{
        unsigned long rebroadcast_time, warning_time;
        int refcnt;
        unsigned long jiffies_save = jiffies;

        linkwatch_forget_dev(dev);

        rebroadcast_time = warning_time = jiffies;
        refcnt = netdev_refcnt_read(dev);

        while (refcnt != 0) {
                if (time_after(jiffies, rebroadcast_time + 1 * HZ)) {
                        rtnl_lock();

                        /* Rebroadcast unregister notification */
                        call_netdevice_notifiers(NETDEV_UNREGISTER, dev);

                        __rtnl_unlock();
                        rcu_barrier();
                        rtnl_lock();

                        call_netdevice_notifiers(NETDEV_UNREGISTER_FINAL, dev);
                        if (test_bit(__LINK_STATE_LINKWATCH_PENDING,
                                     &dev->state)) {
                                linkwatch_run_queue();
                        }

                        __rtnl_unlock();

                        rebroadcast_time = jiffies;
                }

                msleep(250);

                refcnt = netdev_refcnt_read(dev);

                if (refcnt && time_after(jiffies, warning_time + 10 * HZ)) {
                        pr_emerg("unregister_netdevice: waiting for %s to become free. Usage count = %d\n",
                                 dev->name, refcnt);
                        warning_time = jiffies;
                }
                if(time_after(jiffies, jiffies_save + 180 * HZ)){ /* 3 minutes */
                        pr_emerg("BUG: netdev_wait_allrefs: refcnt not release and stuck in dead loop!\n");
                        panic("softlockup: netdev_wait_allrefs hung tasks");
                }
        }
}

```

这个函数会不停的广播 设备的NETDEV_UNREGISTER 通知。在超过180秒后如果索引还是不为0，就触发PANIC。
在crash之前系统一直打印 unregister_netdevice: waiting for vlan2772 to become free. Usage count = 1；
也就是说vlan2772的索引不为0最终导致了这个问题。

## 查看异常栈
用crash工具调试coredump，发现是删除bonding接口，然后释放设备时产生的问题。

```
crash> bt
PID: 3179     TASK: ffff88885ae9a900  CPU: 14   COMMAND: "bsvr"
 #0 [ffffc900011a7b50] machine_kexec at ffffffff80240628
 #1 [ffffc900011a7ba0] __crash_kexec at ffffffff802d19ab
 #2 [ffffc900011a7d30] rtnl_unlock at ffffffff80839e5e
 #3 [ffffc900011a7d40] bonding_store_bonds.cold.3 at ffffffff8067c8ac  
 #4 [ffffc900011a7d90] class_attr_store at ffffffff805f5f2e
 #5 [ffffc900011a7da0] sysfs_kf_write at ffffffff803fde5b
 #6 [ffffc900011a7db0] kernfs_fop_write at ffffffff803fd256
 #7 [ffffc900011a7df0] __vfs_write at ffffffff8038893b
 #8 [ffffc900011a7e70] vfs_write at ffffffff80388beb
 #9 [ffffc900011a7ea8] sys_write at ffffffff80388e1d
#10 [ffffc900011a7ee0] do_syscall_64 at ffffffff80201a5f
crash> dis -l ffffffff8067c8ac
/source/linux-4.14/drivers/net/bonding/bond_sysfs.c: 139    
0xffffffff8067c8ac <bonding_store_bonds.cold.3+67>:     movslq %r14d,%rax

```
对应代码：
```c
static ssize_t bonding_store_bonds(struct class *cls,
                                   struct class_attribute *attr,
                                   const char *buffer, size_t count)
{
//省略无关
        if (command[0] == '-') {
                struct net_device *bond_dev;

                rtnl_lock();
                bond_dev = bond_get_by_name(bn, ifname);
                if (bond_dev) {
                        pr_info("%s is being deleted...\n", ifname);
                        unregister_netdevice(bond_dev);
                } else {
                        pr_err("unable to delete non-existent %s\n", ifname);
                        res = -ENODEV;
                }
                rtnl_unlock();
        }
//省略无关
}
```

因为没有寄存器的值，这给分析带来一些困难。我们从栈信息中找到删除的是lbdp_core这个bonding设备，进而找到设备的per cpu索引地址。
同时也找到了 vlan2772的地址

```

crash> rd ffff88885a0b24b0 2
ffff88885a0b24b0:  6f635f7064626c2d 0000000001006572   -lbdp_core......
crash> net_device.name,pcpu_refcnt ffff88885ae13000  //设备地址
  name = "lbdp_core\000\000\000\000\000\000",
  pcpu_refcnt = 0x6077a040a11c,

crash> net_device.name,pcpu_refcnt -x ffff888828603000
  name = "vlan2772\000\067\067\062\000\000\000",
  pcpu_refcnt = 0x6077a040a188,

```

检查下vlan2772的refcnt值。这里需要特别注意，内核中所以带__percpu的指针，它的地址不能直接读取，需要配合每个cpu的offset，合起来得到一个可读的地址。
```
int __percpu            *pcpu_refcnt;
```
对与基础类型如int, 我们不能方便的读取其per cpu的值。普通的办法是先取得每个cpu的offset值，在通过offset+pcpu_refcnt值得到一个指针，最终得到对应的value。
但是这样太慢。我们可以通过一个单变量结构来方便的读取。例如int型，可以用atomic_t结构来读取。

```
crash> net_device.name,pcpu_refcnt -x ffff888828603000
  name = "vlan2772\000\067\067\062\000\000\000",
  pcpu_refcnt = 0x6077a040a188,
crash> atomic_t 0x6077a040a188:a
[0]: ffffe8ffff60a188
struct atomic_t {
  counter = -7
}
[1]: ffffe8ffff68a188
struct atomic_t {
  counter = -1
}
[2]: ffffe8ffff70a188
struct atomic_t {
  counter = 5
}
[3]: ffffe8ffff78a188
struct atomic_t {
  counter = 0
}
[4]: ffffe8ffff80a188
struct atomic_t {
  counter = 0
}
[5]: ffffe8ffff88a188
struct atomic_t {
  counter = 0
}
[6]: ffffe8ffff90a188
struct atomic_t {
  counter = 0
}
[7]: ffffe8ffff98a188
struct atomic_t {
  counter = 9
}
[8]: ffffe8ffffa0a188
struct atomic_t {
  counter = 0
}
[9]: ffffe8ffffa8a188
struct atomic_t {
  counter = -1
}
[10]: ffffe8ffffb0a188
struct atomic_t {
  counter = 1
}
[11]: ffffe8ffffb8a188
struct atomic_t {
  counter = -5
}
[12]: ffffe8ffffc0a188
struct atomic_t {
  counter = 0
}
[13]: ffffe8ffffc8a188
struct atomic_t {
  counter = -1
}
[14]: ffffe8ffffd0a188
struct atomic_t {
  counter = 0
}
[15]: ffffe8ffffd8a188
struct atomic_t {
  counter = 1
}

```
refcnt汇总起来刚好为1，和系统日志打印一致。

删除bonding lbdp_core 为什么导致vlan2772的释放？ 这是因为vlan2772是建立在 bonding接口lbdp_core上，会被连带释放掉。
释放lbdp_core会调用  call_netdevice_notifiers(NETDEV_UNREGISTER, dev);
而vlan_device_event监听这个事件， 触发对应vlan的unregister。

```c
static int vlan_device_event(struct notifier_block *unused, unsigned long event,
                             void *ptr)
{
        struct net_device *dev = netdev_notifier_info_to_dev(ptr);
        struct vlan_group *grp;
        struct vlan_info *vlan_info;
        int i, flgs;
        struct net_device *vlandev;
        struct vlan_dev_priv *vlan;
        bool last = false;
        LIST_HEAD(list);

        if (is_vlan_dev(dev)) {
                int err = __vlan_device_event(dev, event);

                if (err)
                        return notifier_from_errno(err);
        }

        if ((event == NETDEV_UP) &&
            (dev->features & NETIF_F_HW_VLAN_CTAG_FILTER)) {
                pr_info("adding VLAN 0 to HW filter on device %s\n",
                        dev->name);
                vlan_vid_add(dev, htons(ETH_P_8021Q), 0);
        }
        if (event == NETDEV_DOWN &&
            (dev->features & NETIF_F_HW_VLAN_CTAG_FILTER))
                vlan_vid_del(dev, htons(ETH_P_8021Q), 0);

        vlan_info = rtnl_dereference(dev->vlan_info);
        if (!vlan_info)
                goto out;
        grp = &vlan_info->grp;

        /* It is OK that we do not hold the group lock right now,
         * as we run under the RTNL lock.
         */

        switch (event) {
//省略无关
        case NETDEV_UNREGISTER:
                /* twiddle thumbs on netns device moves */
                if (dev->reg_state != NETREG_UNREGISTERING)
                        break;

                vlan_group_for_each_dev(grp, i, vlandev) {
                        /* removal of last vid destroys vlan_info, abort
                         * afterwards */
                        if (vlan_info->nr_vids == 1)
                                last = true;

                        unregister_vlan_dev(vlandev, &list);  //释放建立在dev上的所有vlan
                        if (last)
                                break;
                }
                unregister_netdevice_many(&list);
                break;

        case NETDEV_PRE_TYPE_CHANGE:
                /* Forbid underlaying device to change its type. */
                if (vlan_uses_dev(dev))
                        return NOTIFY_BAD;
                break;

        case NETDEV_NOTIFY_PEERS:
        case NETDEV_BONDING_FAILOVER:
        case NETDEV_RESEND_IGMP:
                /* Propagate to vlan devices */
                vlan_group_for_each_dev(grp, i, vlandev)
                        call_netdevice_notifiers(event, vlandev);
                break;
        }

out:
        return NOTIFY_DONE;
}

```
从coredump中我们也可以看到 vlan2772的priv中real_dev 真实设备是lbdp_core。

```
crash> vlan_dev_priv -x ffff888828603900              
struct vlan_dev_priv {
  nr_ingress_mappings = 0x0,
  ingress_priority_map = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
  nr_egress_mappings = 0x0,
  egress_priority_map = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
  vlan_proto = 0x81,
  vlan_id = 0xad4,
  flags = 0x1,
  real_dev = 0xffff88885ae13000,  //真实设备是lbdp_core
  real_dev_addr = ".W\324l\t\201",
  dent = 0x0,
  vlan_pcpu_stats = 0x6077a0417e00,
  netpoll = 0x0,
  nest_level = 0x2
}
crash> net_device.name 0xffff88885ae13000 
  name = "lbdp_core\000\000\000\000\000\000",

```

也就是说删除 bonding口 ldbp_core，连带触发 vlan 口 vlan2772的register， 但是有1个索引一直被持有，导致无法释放。
为什么vlan2772会有1个索引没有释放？被谁持有了？
## 查看配置及源码
查看客户配置寻找线索，查找和vlan2772相关的配置，发现相比其他接口多了一个特殊配置 dedicate-management enable。直觉告诉我大概率和这个配置有关。
于是查阅相关代码，发现当我们配置dedicate-management后，会在内核中调用dev_hold增加1个refcnt。看来问题就出在这里了。那么为什么没能正常释放呢？

继续追查代码，发现当你主动删除vlan时，相关逻辑能够判断dedicate-management并删除这个refcnt。这部分判断逻辑在用户态代码中。
然而当你删除bonding口，连带删除相关vlan时，这部分逻辑在内核中，导致dedicate-management的这个refcnt不会被删除。




---
title: "After upgrading the kernel, the IGC network card is not functioning properly"
date: 2025-07-10
---

# 升级内核后，igc网卡无法正常工作

## 1. 问题现象

公司产品内核从linux 4.14.x 升级到 linux 6.1.x 后发现有igc网卡不能正常工作，导致无法联网。

初步观察发现网卡是UP的，但没有RUNNING。

```bash
/# ifconfig port1
Interface @type 1
port1     Link encap:Ethernet  HWaddr 00:03:3D:5D:F9:04  
          inet addr:10.205.250.7  Bcast:0.0.0.0  Mask:255.255.255.255
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Memory:82b00000-82bfffff
/# ethtool -i port1
driver: igc
version: 6.1
firmware-version: 2014:8877
expansion-rom-version: 
bus-info: 0000:09:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes
```

## 2. 初步分析

继续查看发现 port2 也是igc驱动，但是是RUNNING

```bash
/# ifconfig port2
Interface @type 1
port2     Link encap:Ethernet  HWaddr 00:03:3D:5D:F9:05  
          inet6 addr: fe80::203:2dff:fe5d:f905/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10000 
          RX bytes:0 (0.0 B)  TX bytes:600 (600.0 B)
          Memory:82900000-829fffff
/# ethtool -i port2
driver: igc
version: 6.1
firmware-version: 2014:8877
expansion-rom-version: 
bus-info: 0000:0a:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes
```

同样的网卡，同样的驱动，为什么port2可以RUNNING呢？查看协商情况

```bash
/# ethtool  port1
Settings for port1:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	                        2500baseT/Full 
	Supported pause frame use: Symmetric
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  2500baseT/Full   -- 宣告支持速率
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Speed: Unknown!                        -- 速率协商失败
	Duplex: Unknown! (255)
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off (auto)
	Supports Wake-on: pumbg
	Wake-on: g
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: no

/# ethtool  port2
Settings for port2:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	                        2500baseT/Full 
	Supported pause frame use: Symmetric
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  2500baseT/Full   -- 宣告支持速率
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Speed: 2500Mb/s                    ---速率
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off (auto)
	Supports Wake-on: pumbg
	Wake-on: g
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes
```

对比port1和port2 可以看出，问题应该出在 "Advertised link modes:  2500baseT/Full" 这里。
port2协商成功所以RUNNING。 port1和对端协商(对端应该不支持2500bastT/Full)失败导致无法RUNNING。

为了确认猜想，将版本更换成旧版，查看得到。

```bash
/# ifconfig port1
Interface @type 1
port1     Link encap:Ethernet  HWaddr 00:03:3D:5D:F9:04  
          inet addr:10.205.250.7  Bcast:0.0.0.0  Mask:255.255.255.255
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1924 errors:0 dropped:140 overruns:0 frame:0
          TX packets:9220 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10000 
          RX bytes:407648 (398.0 KiB)  TX bytes:13502612 (12.8 MiB)
          Memory:82b00000-82bfffff 


/# ethtool port1
Settings for port1:
	Supported ports: [ ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	                        2500baseT/Full 
	Supported pause frame use: Symmetric
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	                        2500baseT/Full     ---
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Speed: 1000Mb/s
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off (auto)
	Supports Wake-on: pumbg
	Wake-on: g
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes

```

可以发现确实协商模式1000baseT/Full， 之所以能够正常是因为 Advertised link modes包含了。
看到这里基本上就是 Advertised link modes不同导致的。新内核的 Advertised link modes 只支持 2500baseT/Full， 导致协商失败。

## 3. 谁动了 Advertised link modes

为了弄清楚谁动了 Advertised link modes， 我加了debug日志，发现网卡本身的Advertised link modes是支持多种模式的。后续被 igc_ethtool_set_link_ksettings 更改了。
这个函数是用户态ioctl设置的，奇怪的是同样的用户态代码，旧内核就能正常工作。

于是查看函数传入的 ethtool_link_ksettings 结构。

```c
#define DECLARE_BITMAP(name,bits) \
	unsigned long name[BITS_TO_LONGS(bits)]

#define __ETHTOOL_DECLARE_LINK_MODE_MASK(name)		\
	DECLARE_BITMAP(name, __ETHTOOL_LINK_MODE_MASK_NBITS)

struct ethtool_link_ksettings {
	struct ethtool_link_settings base;
	struct {
		__ETHTOOL_DECLARE_LINK_MODE_MASK(supported);
		__ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);
		__ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising);
	} link_modes;
	u32	lanes;
};
```

这是基于 __ETHTOOL_LINK_MODE_MASK_NBITS 来定义无符号长整型数组大小。

突然觉得问题可能和 __ETHTOOL_LINK_MODE_MASK_NBITS 相关，对比新进内核发现
旧内核 linux 4.14.x __ETHTOOL_LINK_MODE_MASK_NBITS 52个bit 一个unsigned long就够了。
新内核 linux 6.1.x  __ETHTOOL_LINK_MODE_MASK_NBITS 93个bit 需要两个unsigned long。

是不是用户态没能适应这个变动导致？

## 4. 用户态 link set操作分析

我们查找到用户态的操作代码

```c
	struct {
		struct ethtool_link_settings req;
		__u32 link_mode_data[3 * SCHAR_MAX];
	} ecmd;

{
//部分代码
	memset(&ecmd,0,sizeof(ecmd));
	ecmd.req.cmd = ETHTOOL_GLINKSETTINGS;
	pIfr->ifr_data = (char*) &ecmd;
	if (ioctl(sockfd, SIOCETHTOOL, pIfr))
	{
		goto END;
	}

	ecmd.req.cmd = ETHTOOL_GLINKSETTINGS;
	ecmd.req.link_mode_masks_nwords = -ecmd.req.link_mode_masks_nwords;
	pIfr->ifr_data = (char*) &ecmd;
	if (ioctl(sockfd, SIOCETHTOOL, pIfr))
	{
		goto END;
	}

	if( autoneg != AUTONEG_ENABLE)
	{
		ecmd.req.speed = speed;
		ecmd.req.duplex = duplex;
	}
	else {
		memcpy(&ecmd.link_mode_data[2],
			&ecmd.link_mode_data[0],
			4 * 2);                          ---这里
		//ecmd.advertising = ecmd.supported;
	}
	ecmd.req.autoneg = autoneg;

	ecmd.req.cmd = ETHTOOL_SLINKSETTINGS;
	int rc = ioctl(sockfd, SIOCETHTOOL, pIfr);
//部分代码
}
```

在 autoneg == AUTONEG_ENABLE 是做了一个hardcode操作， 默认link_mode_data只占 2个 __u32，也就是8个字节。这在linux 4.14.x中是正确的，但是在linux 6.1.x, 是16个字节。
这时 `memcpy(&ecmd.link_mode_data[2],&ecmd.link_mode_data[0],4 * 2)` 操作就会将 advertise前4个字节覆盖成support的中间4个字节。

从debug日志中可以看到：

```bash
[   32.945768] [igc_ethtool_get_link_ksettings:1854] proc:cmdbsvr advertis:af autoneg:af sup:8000000020ef adv:8000000020ef lpad:0

[   32.945770] [igc_ethtool_set_link_ksettings:1868] proc:cmdbsvr advertis:af autoneg:af sup:8000000020ef adv:800000008000 lpad:0
```

从内核中get到的link_mode_data

```
[0x000020ef][0x00008000][0x00000000][0x000020ef][0x00008000][0x00000000]
\------------- support ------------/\------------- advertise ----------/

执行错误的memcpy后
    |-------- 0->2 -----------|
    |                         V
[0x000020ef][0x00008000][0x000020ef][0x00008000][0x00008000][0x00000000]
                  |                         ^
                  |-------- 1->3 -----------|
\------------- support ------------/\------------- advertise ----------/
```

导致advertise中的代表 10baseT/Half 10baseT/Full 100baseT/Half 100baseT/Full 1000baseT/Full 的bit全被覆盖。
从而导致了问题发生。

## 5. 问题修复

问题原因找到了，修复就容易多了。将hardcode部分改为实际link_mode_masks_nwords。

```c
{
	int nwords = ecmd.req.link_mode_masks_nwords;
	memcpy(&ecmd.link_mode_data[nwords],
		&ecmd.link_mode_data[0],
		4 * nwords);
}
```

## 6. 后记

hardcode害人不浅，还是尽量避免。

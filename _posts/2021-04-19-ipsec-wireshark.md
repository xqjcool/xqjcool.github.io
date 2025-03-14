---
title: "How to View IPsec Encrypted Packets in Wireshark"
date: 2021-04-19
---

IPsec 封装安全负载（IPsec ESP）报文，因为是加密封装报文，wireshark打开后无法查看封装内容。

有时候我们需要查看封装的报文内容，以定位一些问题。wireshark支持对ESP的解析，只是需要做一些相应配置。

## 一、抓取并打开ipsec报文
通过tcpdump或者wireshark等方式抓取到ipsec报文。打开后，过滤4500端口(ESP标准端口)，可以看到如下所示封装后的报文。
![image](https://github.com/user-attachments/assets/161a14c5-575f-4d11-8791-3db7ec8e4097)

此时报文会提示是 UDP Encapsulation of IPsec Packets。并告知 ESP SPI: 0xc0a7ebd8 (3232230360)。 ipsec是双向加密，且使用了不同的key，所以反向也有ESP SPI: 0x0caa055e (212469086)。


## 二、获取esp配置
有了SPI，我们就可以在系统上查询相关配置了。
linux系统可以通过ip xfrm state命令获取到esp相应配置。

```bash
[root@5-326-sh-qq-dcpe yl]#  ip xfrm state
src 172.16.23.34 dst 124.64.17.33
  proto esp spi 0x0caa055e reqid 55 mode tunnel
  replay-window 0 flag af-unspec
  auth-trunc hmac(sha256) 0x795cf1a66058a9b7e3004a89417699b815676c0e28d450b4d322b52a3fcfc85b 128
  enc cbc(aes) 0x6580c3bc833a509132b3e2fb32af88fac3cc25984ee2300a32072f7ec0c3a87a
  encap type espinudp sport 4500 dport 19765 addr 0.0.0.0
src 124.64.17.33 dst 172.16.23.34
  proto esp spi 0xc0a7ebd8 reqid 55 mode tunnel
  replay-window 0 flag af-unspec
  auth-trunc hmac(sha256) 0x0448794e814a9250e0e8f6e822b889fe4702d4455b915b5665339f93d59122c5 128
  enc cbc(aes) 0x3fe4c7cf0a73240e02751e1304da3558e8754d633390a11126960eb60477f5e8
  encap type espinudp sport 19765 dport 4500 addr 0.0.0.0
```

通过配置可知

## 三、wireshark上配置esp
wireshark界面操作如下：

### 1.  "Encapsulating Security Payload"右键-->"协议首选项"-->"ESP SAs ..."
![image](https://github.com/user-attachments/assets/6c289a1b-5ddd-469f-9b99-78ab2880b53e)

### 2. ESP SAs界面操作
在ESP SAs界面上点击"+"，创建一个选项如下。
![image](https://github.com/user-attachments/assets/9050934b-270d-4398-ac98-cd6c6453968e)

这里的SrcIP和DstIP自然是wireshark里显示的 192.168.43.102和 49.235.117.117。
SPI： 0xc0a7ebd8
Encryption加密方式: AES-CBC[RFC3602]
加密key：0x6580c3bc833a509132b3e2fb32af88fac3cc25984ee2300a32072f7ec0c3a87a
Authentication：HMAC-SHA-256-128[RFC4868]
Authentication Key：0x795cf1a66058a9b7e3004a89417699b815676c0e28d450b4d322b52a3fcfc85b

同样步骤配置 49.235.117.117-->192.168.43.102 方向的SA。
配置完毕后如下图：
![image](https://github.com/user-attachments/assets/039fd63a-43a8-4fb1-a059-25750194bbe2)

### 3. 解密后的报文
一般来说，这回就可以看到解密后的报文了。
部分wireshark需要显式打开 "Attempt to detect/decode encrypted ESP payloads"选项。
![image](https://github.com/user-attachments/assets/fa108ef7-f1ad-4481-846f-a58338b2c9f1)

然后就可以看到解密后的报文内容，进行相应的查看分析了。
![image](https://github.com/user-attachments/assets/591e8d71-ecd1-4ee2-ace9-292587d6de71)


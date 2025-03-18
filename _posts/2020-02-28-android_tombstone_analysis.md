---
title: "[crash][android]Program Crash Analysis Methods"
date: 2020-02-28
---

	android程序在运行过程中发生异常，会产生tombstone文件，放在/data/tombstones目录下。我们可以通过分析tombstone来定位问题原因。
## 第一步：查看tombstone
	tombstone是文本格式，所以任何文本编辑器都可以打开。我们第一步先过一下tombstone的关键信息。


```
Build fingerprint: 'google/sailfish/sailfish:9/PQ3A.190801.002/5670241:user/release-keys'
Revision: '0'
ABI: 'arm'
pid: 9173, tid: 9179, name: testThread  >>> /data/user/0/com.test/files/testapp <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x4798
    r0  00004780  r1  00000780  r2  00000060  r3  e0cbc018
    r4  e71f01d0  r5  db53b4b0  r6  db53b4a8  r7  00000000
    r8  e71f01d4  r9  e71efa10  r10 e7cd94ec  r11 e16ff848
    ip  b9839850  sp  e16ff830  lr  b951a62c  pc  b9519f10

backtrace:
    #00 pc 000a3f10  /data/data/com.test/files/testapp

stack:
         e16ff7f0  00000000
         e16ff7f4  00000000
         e16ff7f8  00000002
         e16ff7fc  ffffffff
         e16ff800  e0c8f1c0  [anon:libc_malloc]
         e16ff804  e16ff8a8  <anonymous:e1603000>
         e16ff808  e0c83640  [anon:libc_malloc]
         e16ff80c  0000c50b
         e16ff810  e71e6900  [anon:libc_malloc]
         e16ff814  00000000
         e16ff818  e71efa17  [anon:libc_malloc]
         e16ff81c  e71efa18  [anon:libc_malloc]
         e16ff820  dfea3014  [anon:libc_malloc]
         e16ff824  00000000
         e16ff828  acb0ac7c
         e16ff82c  00000000
    #00  e16ff830  00000000
         e16ff834  b98c1694  <anonymous:b983a000>
         e16ff838  00400014
         e16ff83c  00004000
         e16ff840  00004780
         e16ff844  80140000
         e16ff848  e16ff8b8  <anonymous:e1603000>
         e16ff84c  b951a62c  /data/data/com.test/files/testapp
```
	这是testapp程序在testThread中产生的异常。异常原因是SEGV_MAPPER，fault addr 0x4798。很明显这是一个异常地址访问导致的crash。但是仅从这些信息不足以定位到问题出在哪里。
	

## 第二步：addr2line

	这一步就该addr2line出场了。我们从backstrace中拿到了出错时的指令地址，它对应我们程序中某个函数的某行操作。我们需要addr2line将对应文件和对应行数打出来。这个命令的格式是 addr2line -e exefile addr
	在NDK中找到对应的addr2line命令，testapp程序文件取出来，执行命令
	

```powershell
[root@localhost]# /opt/android_tools/android-ndk-r20/toolchains/llvm/prebuilt/linux-x86_64/bin/arm-linux-androideabi-addr2line -f -e testapp 000a3f10
testGetEntryById
/home/***/***/***/test.c:220
```

	于是我们得到了出错的具体位置，对照源码，大多数问题就可以分析出错误原因。
	从代码中看应该是entry出了问题，但是entry是block中的结构取的地址，block在创建时都初始化了，按理不应该出问题。

```c
//代码示例，做了简化，方便理解
struct entry *testGetEntryById(unsigned short Id)
{
    ...
    blockId = Id >> 9;
    blockOffset = Id & (1 << 9 - 1);
    
    block = g_Table.EntryBlock[blockId];
    if (block == NULL)
    {
        return NULL；
    }

    entry = &block->Entry[blockOffset];
    if (!entry->Valid)	//问题出在这一行
    {
        return NULL;
    }
    ...
}
```
	从代码上难以分析出问题原因，下一步就需要查看汇编代码了。
## 第三步：objdump
	为了进一步定位问题，我们需要将出错函数的汇编取到，这就用到了objdump工具。我们使用objdump将testapp反汇编重定向到文件中。然后查看对应函数的汇编代码。
	

```powershell
000a3e54 <testGetEntryById>:
   a3e54:       e92d4800        push    {fp, lr}
   a3e58:       e1a0b00d        mov     fp, sp
   a3e5c:       e24dd018        sub     sp, sp, #24
   a3e60:       e59f10d0        ldr     r1, [pc, #208]  ; a3f38 <testGetEntryById+0xe4>
   a3e64:       e08f1001        add     r1, pc, r1
   a3e68:       e2811004        add     r1, r1, #4
   a3e6c:       e14b00b2        strh    r0, [fp, #-2]
...//省略
   a3f08:       e50b0008        str     r0, [fp, #-8]
   a3f0c:       e51b0008        ldr     r0, [fp, #-8]
   a3f10:       e5900018        ldr     r0, [r0, #24]	//问题发生位置
```
	对照源码和汇编，我们fp-2位置存放了入参Id，我们在tombstone中可以找到fp(即r11)为e16ff848，在stack中可以找到fp-2的值为0x8014。
	

```powershell
//tombstone 截取
         e16ff844  80140000				//fp-4，所以fp-2是0x8014(注意这里是小端字节序)
         e16ff848  e16ff8b8  <anonymous:e1603000>	//fp地址
```
	Id = 32768， 右移9bit后是64,即blockId是64。但是g_Table.EntryBlock[]数组大小为64， index范围 0~63。64超出数组范围，造成访问溢出，取到了错误的block地址。从而导致的最终的问题发生。

## 总结
1. 获取并查看tombstone，获知问题的基本信息
2. 通过addr2line找到问题发生的具体位置。对照源码分析问题原因。
3. 如果任然不能确认问题原因，我们可以对问题函数进行反汇编，对照tombstone中的寄存器，异常栈来模拟当时的执行过程。最终找到问题的根本原因

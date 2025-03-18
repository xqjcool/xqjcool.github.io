---
title: "[Windows] Configuration and Debugging Methods for Generating DMP Files When an Executable Program Encounters an Exception"
date: 2020-03-01
---

正常情况下，可执行程序异常退出后不会生成dmp文件，这给定位分析问题原因带来了极大的困难。通过修改注册表我们可以让程序生成dmp文件，配合windbg工具的使用，可以方便的帮助定位异常问题原因。
## 1. 注册表修改操作

```powershell
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Error Reporting
```

### 1.1 创建LocalDumps注册表项
在"Windows Error Reporting"上右击，新建-->项，然后重命名为"LocalDumps"。

### 1.2 创建可执行程序对应注册表项
假如我们的可执行程序叫wintest.exe，那么我们创建的表项名就叫这个名字。
在"LocalDumps"上右击，新建-->项，重命名为"wintest.exe"。


### 1.3 添加相关属性配置
在左边wintest.exe上右击或者在右边空白处右击，新建-->DWORD,重命名为"DumpCount"，设置值为a，即允许同时存在10个dmp文件。

同样方法，新建-->DWORD,重命名为DumpType，双击该属性，设置值为2，即full dump。
同样方法，新建-->可扩充字符串值，重命名为DumpFolder，双击该属性，设置为程序的所在目录。比如"D:\Program Files\Test"。
配置完成后如下图。

配置完成后，程序异常后就会产生wintest.exe.xxx.dmp文件，供我们后续调试。

## 2. 程序dmp文件的调试方法
这里我们选用大名鼎鼎的windbg工具。
### 2.1 windbg工具安装方法
#### 2.1.1 下载winsdksetup.exe
点击链接，下载winsdksetup.exe。
[winsdksetup.exe](https://download.microsoft.com/download/4/2/2/42245968-6A79-4DA7-A5FB-08C0AD0AE661/windowssdk/winsdksetup.exe)
#### 2.1.2 下载指定组件
运行winsdksetup.exe后，选择"Continue"，指定下载路径，下一步会提示选择需要的功能组件，我们只要选择 "Debugging Tools for Windows"即可。
![image](https://github.com/user-attachments/assets/3e66c4b6-df73-4ba1-b362-7f3fb2b6c3ac)

点击下载，完成后，我们在下载路径可以看到相应的debugger安装文件文件。
![image](https://github.com/user-attachments/assets/42bedeb2-0613-491e-b349-a26897d60b65)

#### 2.1.3 安装windbg
我们wintest.exe是32位的可执行程序，所以我们选择安装X86 Debugger，双击"X86 Debuggers And Tools-x86_en-us.msi"文件。安装完成后，在开始界面会显示对应的程序。
![image](https://github.com/user-attachments/assets/7fbe2cfc-a0b8-4b92-97d6-5dce97e4bfed)


### 2.2 使用windbg调试dmp文件
打开windbg程序，选择 File -> Open Crash Dump ，然后选择对应的 dmp文件，点击打开。
![image](https://github.com/user-attachments/assets/5ac0c075-39d7-4340-8ca0-ee4fb0815185)

加载dmp文件后，就会出现Command窗口。就可以开始调试分析问题了。
![image](https://github.com/user-attachments/assets/d82bfd0a-1c41-4c1b-929d-f9f9124f814a)




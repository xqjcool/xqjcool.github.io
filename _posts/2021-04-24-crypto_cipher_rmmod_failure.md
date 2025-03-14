---
title: "[Module Unloading Failure] Business Module Fails to Unload Due to Encryption Algorithm Issue."
date: 2021-04-24
---

业务模块使用了内核的crypto API来对报文进行加解密。之前使用的是内核自带的 aes，des加解密方式。一切正常，模块的加载卸载也OK。(基于CentOS 3.10.0-514版本)

最近为了符合国家密码标准，增加了sm4加解密算法。并在业务模块初始化时，调用crypto_register_alg将sm4算法注册起来。

然后使用 crypto_alloc_cipher 创建加密cipher，crypto_cipher_setkey设置加密key。调用 crypto_cipher_encrypt_one 和 crypto_cipher_decrypt_one对报文进行加解密操作。

卸载模块时会crypto_free_cipher 释放cipher，并调用 crypto_unregister_alg注销算法。

看起来很合理。但是在卸载模块时出现了问题。模块无法卸载。

查看模块信息，发现索引值为1。

```bash
[root@localhost ~]# lsmod | grep bsnmod
bsnmod               3210005  1 
[root@localhost ~]# rmmod bsnmod
rmmod: ERROR: Module bsnmod is in use
```

内核卸载模块时会检查mod的 refcnt，如果不为0，不允许卸载。

```c
SYSCALL_DEFINE2(delete_module, const char __user *, name_user,
		unsigned int, flags)
{
//...
	/* Never wait if forced. */
	if (!forced && module_refcount(mod) != 0)
		wait_for_zero_refcount(mod);
//...
}
```

那这个索引哪里来的呢。 正常情况下是0的。因为是添加了sm4算法后才这样的，我们自然就把怀疑点放在了sm4的相关操作上。

然后就发现
crypto_alloc_cipher--> crypto_alloc_base-->crypto_alg_mod_lookup-->crypto_larval_lookup-->crypto_alg_lookup-->__crypto_alg_lookup-->crypto_mod_get-->try_module_get-->增加模块索引

会增加算法alg所属模块的引用。而我们alg所属模块就是业务模块bsnmod。也就是说bsnmod的索引＋1。

就是它导致了模块卸载失败。

究其原因，就是我们实现的sm4算法，crypto_register_alg注册时算法的模块是bsnmod业务模块，这样我们bsnmod模块中使用crypto_alloc_cipher 申请该算法的cipher时，会导致sm4算法模块的引用计数增加，也就是bsnmod的引用计数增加。

卸载模块时，虽然我们在exit函数中有 crypto_free_cipher 释放和crypto_unregister_alg注销操作。但是执行exit之前会有 模块引用计数判断，如果计数不为0，则直接报错退出，导致无法卸载模块。

看来sm4的算法不能放在bsnmod中进行使用，为了规避这个问题，sm4算法必须单独作为一个模块加载。

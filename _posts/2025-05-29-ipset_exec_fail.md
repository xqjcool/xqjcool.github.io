---
title: "ipset command fails to execute after upgrading the kernel"
date: 2025-05-29
---

# 升级内核后，ipset执行失败的问题分析

## 1. 问题现象

这段时间负责产品内核升级。由于内核升级跨度大，冲突特别多，代码特别乱。
终于内核能够稳定运行后，测试发现ipset不能正常执行。

## 2. 初步分析

初步分析发现新内核ipset版本升级到7了，而应用程序ipset 6.38 支持版本还是6。
于是下载了最新的ipset-7.24, 由于我们的ipset是定制过的，添加了一些自定义字段。所以还得将旧代码做diff生成patch，再应用打新ipset中，修复冲突。

一切完成之后，编译版本。本以为万事大吉，没想到运行失败。

```bash
ipset v7.24: Kernel error received: ipset protocol error
```

## 3. 继续深入

### 3.1 ipset debug

为了方便定位，将ipset的debug开关打开了

```bash
#define IPSET_DEBUG 1  //开启debug打印
```

### 3.2 kernel ipset debug

同时内核中将可能返回 `-IPSET_ERR_PROTOCOL` 的地方都加了打印。

### 3.3 调试迭代

先是发现是 `ip_set_ad` 这个函数返回的 `-IPSET_ERR_PROTOCOL` 。

```bash
May 29 10:31:19 kern.info kernel: [  157.433117] [ip_set_ad:2017]
```

对应函数下面部分：

```c
static int ip_set_ad(struct net *net, struct sock *ctnl,
		     struct sk_buff *skb,
		     enum ipset_adt adt,
		     const struct nlmsghdr *nlh,
		     const struct nlattr * const attr[],
		     struct netlink_ext_ack *extack)
{
//省略无关

	set = find_set(inst, nla_data(attr[IPSET_ATTR_SETNAME]));
	if (!set)
		return -ENOENT;

	use_lineno = !!attr[IPSET_ATTR_LINENO];
	if (attr[IPSET_ATTR_DATA]) {
		if (nla_parse_nested(tb, IPSET_ATTR_ADT_MAX,
				     attr[IPSET_ATTR_DATA],
				     set->type->adt_policy, NULL)) {
				     
			pr_info("[%s:%d] \n", __func__, __LINE__);    //这里
			return -IPSET_ERR_PROTOCOL;
				     	}
		ret = call_ad(net, ctnl, skb, set, tb, adt, flags,
			      use_lineno);
	} else {
//省略无关
	}
	return ret;
}
```

也就是说是 `nla_parse_nested(tb, IPSET_ATTR_ADT_MAX, attr[IPSET_ATTR_DATA], set->type->adt_policy, NULL)` 处理过程中出错导致了命令执行失败。
也就是说在分析嵌套的 `IPSET_ATTR_DATA` nla 内容时有错误发生。
找到突破点后，继续在 `nla_parse_nested` 中 添加调试日志， 编译，运行后，可以清晰发现错误位置。

- ipset命令的debug 输出

```bash
 # ipset add ip_set_gp1 203.203.203.203,udp:65535 priv1 1
//省略无关部分
Message header: sent cmd  ADD (9) flags: 0x205
	len 92
	flag none
	seq 1748540363
	Command attributes:
	PROTOCOL: 7
	SETNAME: ip_set_gp1
		ADT attributes:
		IP: 203.203.203.203
		PORT: 65535
		PROTO: 17
		CADT_LINENO: 0
		PRIV1: 1
Message header: received msg ERROR, flags: 0
	len 112
	errcode -4097
	seq 1748540363
session.c: callback_error:  called, cmd ADD
session.c: callback_error: nlmsgerr error: 4097
session.c: decode_errmsg: nlsmg_len: 92
session.c: generic_data_attr_cb: attr type: 1, len 5
session.c: generic_data_attr_cb: attr type: 2, len 15
session.c: generic_data_attr_cb: attr type: 7, len 48
mnl.c: ipset_mnl_query: nfln_cb_run2, ret: -1, errno 0
session.c: ipset_commit: ret: -1
session.c: ipset_cmd: reset data
ipset.c: ipset_parse_argv: ret -1
ipset v7.24: Kernel error received: ipset protocol error
ipset.c: default_custom_error: status: 4
```

- 内核日志部分

```bash
May 29 10:39:20  kern.info kernel: [  638.073733] [__nla_validate_parse:653] OK type:1
May 29 10:39:20  kern.info kernel: [  638.073836] [__nla_validate_parse:653] OK type:1
May 29 10:39:20  kern.info kernel: [  638.073840] [__nla_validate_parse:653] OK type:2
May 29 10:39:20  kern.info kernel: [  638.074255] [__nla_validate_parse:653] OK type:1
May 29 10:39:20  kern.info kernel: [  638.074260] [__nla_validate_parse:653] OK type:2
May 29 10:39:20  kern.info kernel: [  638.074263] [__nla_validate_parse:653] OK type:7
May 29 10:39:20  kern.info kernel: [  638.074266] [__nla_validate_parse:653] OK type:1
May 29 10:39:20  kern.info kernel: [  638.074268] [__nla_validate_parse:653] OK type:4
May 29 10:39:20  kern.info kernel: [  638.074271] [__nla_validate_parse:653] OK type:7
May 29 10:39:20  kern.info kernel: [  638.074273] [validate_nla:545] 
May 29 10:39:20  kern.info kernel: [  638.074277] [__nla_validate_parse:646]type:12 err:-22
May 29 10:39:20  kern.info kernel: [  638.074281] [ip_set_ad:2017] 
```

### 3.4 代码分析

从内核日志上看， 是 `__nla_validate_parse` 中 调用 `validate_nla` 去验证 nla时出错导致。

```c
static int __nla_validate_parse(const struct nlattr *head, int len, int maxtype,
				const struct nla_policy *policy,
				unsigned int validate,
				struct netlink_ext_ack *extack,
				struct nlattr **tb, unsigned int depth)
{
//省略无关

	nla_for_each_attr(nla, head, len, rem) {
		u16 type = nla_type(nla);

		if (type == 0 || type > maxtype) {
			if (validate & NL_VALIDATE_MAXTYPE) {
				NL_SET_ERR_MSG_ATTR(extack, nla,
						    "Unknown attribute type");
				pr_info("[%s:%d] \n", __func__, __LINE__);
				return -EINVAL;
			}
			continue;
		}
		type = array_index_nospec(type, maxtype + 1);
		if (policy) {
			int err = validate_nla(nla, maxtype, policy,
					       validate, extack, depth);

			if (err < 0) {
				pr_info("[%s:%d]type:%u err:%d\n", __func__, __LINE__, type, err);  //这里
				return err;
				}
		}

//省略无关
	return 0;
}

static int validate_nla(const struct nlattr *nla, int maxtype,
			const struct nla_policy *policy, unsigned int validate,
			struct netlink_ext_ack *extack, unsigned int depth)
{
//省略无关

	switch (pt->type) {
//省略无关
	case NLA_UNSPEC: 
		if (validate & NL_VALIDATE_UNSPEC) {
			NL_SET_ERR_MSG_ATTR(extack, nla,
					    "Unsupported attribute");
			pr_info("[%s:%d] \n", __func__, __LINE__);  //这里
			return -EINVAL;
		}
		if (attrlen < pt->len) {
			pr_info("[%s:%d] \n", __func__, __LINE__);
			goto out_err;
			}
		break;

	default:
		if (pt->len)
			minlen = pt->len;
		else
			minlen = nla_attr_minlen[pt->type];

		if (attrlen < minlen) {
			pr_info("[%s:%d] \n", __func__, __LINE__);
			goto out_err;
			}
	}
//省略无关

	return 0;
out_err:
	NL_SET_ERR_MSG_ATTR_POL(extack, nla, pt,
				"Attribute failed policy validation");
	return err;
}
```

从log上知道 验证的是type=12，也就是我们自定义的PRIV1 参数。从代码上看是进入了 NLA_UNSPEC 的处理。为了知道 type=12对应的policy，我们使用crash工具实时调试内核。

```bash
crash> ip_set_net_id
ip_set_net_id = $1 = 35
crash> p init_net.gen.ptr[35]
$2 = (void *) 0xffff88810195e6d0
crash> ip_set_net -x 0xffff88810195e6d0 
struct ip_set_net {
  ip_set_list = 0xffff88810e8cf800,
  ip_set_max = 0x100,
  is_deleted = 0x0,
  is_destroyed = 0x0
}
crash> ip_set_net -x
struct ip_set_net {
    struct ip_set **ip_set_list;
    ip_set_id_t ip_set_max;
    bool is_deleted;
    bool is_destroyed;
}
SIZE: 0x10
crash> rd 0xffff88810e8cf800 6
ffff88810e8cf800:  ffff888101444d80 ffff888112f22fc0   .MD....../......
ffff88810e8cf810:  0000000000000000 0000000000000000   ................
ffff88810e8cf820:  0000000000000000 0000000000000000   ................

crash> ip_set ffff888112f22fc0
struct ip_set {
  name = "ip_set_gp1\000",
  lock = {
    {
      rlock = {
        raw_lock = {
          {
            val = {
              counter = 0
            },
            {
              locked = 0 '\000',
              pending = 0 '\000'
            },
            {
              locked_pending = 0,
              tail = 0
            }
          }
        },
        magic = 3735899821,
        owner_cpu = 4294967295,
        owner = 0xffffffffffffffff
      }
    }
  },
  ref = 0,
  ref_netlink = 0,
  type = 0xffffffff81e6fe20 <hash_ipport_type>,
  variant = 0xffffffff8198be20 <hash_ipport4_variant>,
  family = 2 '\002',
  revision = 6 '\006',
  extensions = 0 '\000',
  flags = 2 '\002',
  timeout = 4294967295,
  elements = 0,
  ext_size = 0,
  dsize = 16,
  offset = {0, 0, 0, 0},
  data = 0xffff888112f23080
}

crash> 
crash> p hash_ipport_type.adt_policy[12]
$6 = {
  type = 0 '\000',          //NLA_UNSPEC
  validation_type = 0 '\000',
  len = 0,
  {
    strict_start_type = 0,
    bitfield32_valid = 0,
    mask = 0,
    reject_message = 0x0,
    nested_policy = 0x0,
    range = 0x0,
    range_signed = 0x0,
    {
      min = 0,
      max = 0
    },
    validate = 0x0
  }
}

```

`policy[12]`的的type确实是 NLA_UNSPEC， 那么传入的validate呢？ 我们查到是 `NL_VALIDATE_STRICT` 

```c
#define NL_VALIDATE_STRICT (NL_VALIDATE_TRAILING |\
			    NL_VALIDATE_MAXTYPE |\
			    NL_VALIDATE_UNSPEC |\
			    NL_VALIDATE_STRICT_ATTRS |\
			    NL_VALIDATE_NESTED)

static inline int nla_parse_nested(struct nlattr *tb[], int maxtype,
				   const struct nlattr *nla,
				   const struct nla_policy *policy,
				   struct netlink_ext_ack *extack)
{
	if (!(nla->nla_type & NLA_F_NESTED)) {
		NL_SET_ERR_MSG_ATTR(extack, nla, "NLA_F_NESTED is missing");
		pr_info("[%s:%d] \n", __func__, __LINE__);
		return -EINVAL;
	}

	return __nla_parse(tb, maxtype, nla_data(nla), nla_len(nla), policy,
			   NL_VALIDATE_STRICT, extack);
}
```

所以在 `validate_nla` 中会被认为错误而返回。

### 3.5 根因复现

基于以上就是policy中没有自定义PRIV1的policy， 导致type为 NLA_UNSPEC， 从而失败。
检查代码，确实没有 type=12 即 IPSET_ATTR_PRIVATE1 的policy，这是导致问题的**根本原因**。

```c
static struct ip_set_type hash_ipport_type __read_mostly = {
	.name		= "hash:ip,port",
	.protocol	= IPSET_PROTOCOL,
	.features	= IPSET_TYPE_IP | IPSET_TYPE_PORT,
	.dimension	= IPSET_DIM_TWO,
	.family		= NFPROTO_UNSPEC,
	.revision_min	= IPSET_TYPE_REV_MIN,
	.revision_max	= IPSET_TYPE_REV_MAX,
	.create_flags[IPSET_TYPE_REV_MAX] = IPSET_CREATE_FLAG_BUCKETSIZE,
	.create		= hash_ipport_create,
	.create_policy	= {
		[IPSET_ATTR_HASHSIZE]	= { .type = NLA_U32 },
		[IPSET_ATTR_MAXELEM]	= { .type = NLA_U32 },
		[IPSET_ATTR_INITVAL]	= { .type = NLA_U32 },
		[IPSET_ATTR_BUCKETSIZE]	= { .type = NLA_U8 },
		[IPSET_ATTR_RESIZE]	= { .type = NLA_U8  },
		[IPSET_ATTR_PROTO]	= { .type = NLA_U8 },
		[IPSET_ATTR_TIMEOUT]	= { .type = NLA_U32 },
		[IPSET_ATTR_CADT_FLAGS]	= { .type = NLA_U32 },
	},
	.adt_policy	= {
		[IPSET_ATTR_IP]		= { .type = NLA_NESTED },
		[IPSET_ATTR_IP_TO]	= { .type = NLA_NESTED },
		[IPSET_ATTR_PORT]	= { .type = NLA_U16 },
		[IPSET_ATTR_PORT_TO]	= { .type = NLA_U16 },
		[IPSET_ATTR_CIDR]	= { .type = NLA_U8 },
		[IPSET_ATTR_PROTO]	= { .type = NLA_U8 },
		[IPSET_ATTR_TIMEOUT]	= { .type = NLA_U32 },
		[IPSET_ATTR_LINENO]	= { .type = NLA_U32 },
		[IPSET_ATTR_BYTES]	= { .type = NLA_U64 },
		[IPSET_ATTR_PACKETS]	= { .type = NLA_U64 },
		[IPSET_ATTR_COMMENT]	= { .type = NLA_NUL_STRING,
					    .len  = IPSET_MAX_COMMENT_SIZE },
		[IPSET_ATTR_SKBMARK]	= { .type = NLA_U64 },
		[IPSET_ATTR_SKBPRIO]	= { .type = NLA_U32 },
		[IPSET_ATTR_SKBQUEUE]	= { .type = NLA_U16 },
	},
	.me		= THIS_MODULE,
};
```

那么为什么升级前是好的呢？
对比代码发现，旧内核代码并没有检查 type为 NLA_UNSPEC的policy，也就是忽略了。而新内核检查更严格，有未在policy中定义的，则会判定为错误。

## 4. 总结

这次升级内核遇到了好多因为检查不通过而失败的问题了。归根结底是内核为了安全，检查越来越严格了。


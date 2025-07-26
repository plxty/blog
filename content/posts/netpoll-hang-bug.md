+++
title = "埋藏的 Netpoll 问题"
date = "2025-07-26"

[taxonomies]
tags = ["try-catch"]
+++

# Catch

这是一个很神奇的 bug，我作为“victim”有幸围观了整个问题发现以及修复的过程，算是对 Linux 社区的运作方式多了一点参与（感）。

起因是在社区合入了我 [virtio-net 死锁修复](https://github.com/torvalds/linux/commit/be5dcaed694e4255dc02dd0acfe036708c535def)的提交之后，发现 NAPI 测试有一定概率会 hang 住，也就顺理成章的[怀疑](https://lore.kernel.org/netdev/20250722145524.7ae61342@kernel.org/)到了我这个 patch 上，

```bash
[  370.088243][   T44] Showing all locks held in the system:
[  370.088588][   T44] 3 locks held by kworker/u16:0/12:
[  370.088843][   T44]  #0: ffff88800933e548 ((wq_completion)ipv6_addrconf){+.+.}-{0:0}, at: process_one_work+0x7e5/0x1660
[  370.089314][   T44]  #1: ffffc900000c7d40 ((work_completion)(&(&net->ipv6.addr_chk_work)->work)){+.+.}-{0:0}, at: process_one_work+0xdf6/0x1660
[  370.089907][   T44]  #2: ffffffffa9456648 (rtnl_mutex){+.+.}-{4:4}, at: addrconf_verify_work+0x12/0x30
[  370.090318][   T44] 1 lock held by khungtaskd/44:
[  370.090549][   T44]  #0: ffffffffa8b796e0 (rcu_read_lock){....}-{1:3}, at: debug_show_all_locks+0x36/0x260
[  370.090959][   T44] 3 locks held by kworker/2:1/62:
[  370.091181][   T44]  #0: ffff8880010a9548 ((wq_completion)events){+.+.}-{0:0}, at: process_one_work+0x7e5/0x1660
[  370.091629][   T44]  #1: ffffc90000437d40 ((work_completion)(&vi->rx_mode_work)){+.+.}-{0:0}, at: process_one_work+0xdf6/0x1660
[  370.092146][   T44]  #2: ffffffffa9456648 (rtnl_mutex){+.+.}-{4:4}, at: virtnet_rx_mode_work+0x145/0x860
[  370.092547][   T44] 2 locks held by ip/834:
[  370.092732][   T44]  #0: ffffffffa9456648 (rtnl_mutex){+.+.}-{4:4}, at: rtnl_newlink+0x651/0xa60
[  370.093132][   T44]  #1: ffff888008e4ac80 (&dev->lock){+.+.}-{4:4}, at: napi_disable+0x3b/0x80
```

完整的日志可以在[这里](https://netdev-3.bots.linux.dev/vmksft-drv-hw-dbg/results/209441/4-stats-py/stderr)找到，分析之后找到具体卡死的地方在于 `napi_disable()` 时会有个忙等，等待所有其他 NAPI 完成他们的工作（基本就是 `napi->poll()`），

[//]: # (感觉 zola 的 codeblock 还可以，支持很多小特性，可以参考 https://github.com/getzola/zola/blob/45d3f8d6285f0b47013c5fa31eb405332118af8b/components/markdown/src/codeblock/fence.rs#L106 里列出的这些属性。)

[//]: # (另外这居然可以是一种注释，来源 https://zola.discourse.group/t/tera-comments-in-markdown/450/2)

```c,hl_lines=6-9
void napi_disable_locked(struct napi_struct *n)
{
	// ----- snippet -----
	val = READ_ONCE(n->state);
	do {
		while (val & (NAPIF_STATE_SCHED | NAPIF_STATE_NPSVC)) {
			usleep_range(20, 200);
			val = READ_ONCE(n->state);
		}

		new = val | NAPIF_STATE_SCHED | NAPIF_STATE_NPSVC;
		new &= ~(NAPIF_STATE_THREADED | NAPIF_STATE_PREFER_BUSY_POLL);
	} while (!try_cmpxchg(&n->state, &val, new));
	// ----- snippet -----
}
```

另外这个 CI 是来自社区维护的 [NIPA](https://github.com/linux-netdev/nipa)，其中用了 virtme-ng 等方式来拉起虚拟机跑 kselftests。

# Reproduce

因为 Jakub Kicinski 第一眼觉得是我这个 patch 引入的问题，因为以前从来没发生过，所以机缘巧合之下我就被“拉近了”这个讨论中，

```
From: Jakub Kicinski <kuba@kernel.org>
To: Paolo Abeni <pabeni@redhat.com>, Zigit Zo <zuozhijie@bytedance.com>
```

（只有当大佬觉得你可能闯祸了的时候才会来主动联系你😶‍🌫️）

但是我自己，还有 virtio-net 的一位 maintainer Jason Wang 也表示难以复现，后来我看了其他的一些测试结果才发现，在某个时间节点后，引入了一个额外的 `netpoll_basic.py` 测试脚本，而这个节点刚好是我合入 patch 的那附近，所以实际上这个 bug 会不会是因为这个测试脚本测出来的，而非我引入的？

当我回了个邮件表达完自己疑惑后，就开始重新搭环境想把整个 NIPA 跑起来，然鹅还在我搭环境的时候 Jakub Kicinski 就已经[找到了问题](https://lore.kernel.org/netdev/20250726010846.1105875-1-kuba@kernel.org)并发了个新的 patch。

# Cause

最后发现问题确实是跟新引入的测试脚本有关，它揭露了一个埋藏了很久很久的问题。

修复的 patch 里的这个 `Fixes: 1da177e4c3f4 ("Linux-2.6.12-rc2")` 实际上时 Linux 刚迁移到 git 时的第一个提交，因此真正的出现时间应该更早，当然，起码得是 RCU 引入之后。

这就是一个很典型的 data race 了，是关于 netpoll 和 NAPI 之间的竞争。

## Netpoll

在了解这个 bug 前，我们可能得知道 netpoll 是个什么东西，看上面也知道，这个机制在内核已经存在相当长的时间了，引入的目的在于应付一些极端的系统情况，例如当系统 panic 后，中断处理函数都失效了。这个需求就决定了 netpoll 的实现会绕过整个网络协议栈（例如 netfilter 等），直接到网卡驱动层收发包，例如收包就是通过直接 loop `napi->poll()` 来完成的。

发现一篇[介绍 netpoll](https://chengqian90.com/Linux%E5%86%85%E6%A0%B8/Linux-Netpoll%E6%B5%85%E6%9E%90.html) 的文章写的挺好的，推荐看一看。

上面提到的 `netpoll_basic.py` 脚本则是使用了 netconsole，就是转发 kmsg 到远端的机制，很多远程监控组件例如 syslogd 等都支持配置，而它正是使用了 netpoll 这个机制完成的。脚本本身很好理解，

0. 需要环境配置好对端 VM，这样才能实现互通测试，即 host 也需要启动一些服务，NIPA 似乎会帮你完成
1. 启动一个 iperf3 实例进行 NAPI 收包
2. 配置好 netconsole，不断往 `/dev/kmsg` 写入内容，触发 netconsole 发送
3. 通过 bpftrace 钩住 netpoll 接口，来看 netpoll 的调用次数等是否正确

## Race

大概了解 netpoll 后，就可以来看一下这个 bug 了，实际上发生 race 的是这两个地方，

```c,hl_lines=6
static void poll_one_napi(struct napi_struct *napi)
{
	// ----- snippet -----
	if (test_and_set_bit(NAPI_STATE_NPSVC, &napi->state))
		return;
	work = napi->poll(napi, 0);
	clear_bit(NAPI_STATE_NPSVC, &napi->state);
	// ----- snippet -----
}
```

```c,hl_lines=6 10
bool napi_complete_done(struct napi_struct *n, int work_done)
{
	// ----- snippet -----
	if (unlikely(n->state & (NAPIF_STATE_NPSVC |
				 NAPIF_STATE_IN_BUSY_POLL)))
		return false;

	val = READ_ONCE(n->state);
	do {
		new = val & ~(NAPIF_STATE_MISSED | NAPIF_STATE_SCHED |
			      NAPIF_STATE_SCHED_THREADED |
			      NAPIF_STATE_PREFER_BUSY_POLL);
		new |= (val & NAPIF_STATE_MISSED) / NAPIF_STATE_MISSED *
						    NAPIF_STATE_SCHED;
	} while (!try_cmpxchg(&n->state, &val, new));
	// ----- snippet -----
}
```

最主要的就是 `NAPI_STATE_NPSVC` 和 `NAPI_STATE_SCHED` 这两个标志了，上面提到的 `napi_disable_locked()` 就会不断尝试读取这两个标志位，当它们任意一个被置位是就表示这个 NAPI 此时还有人在占用，无法 disable，就会一直等（而当它等到没有人占用之后就会自己设置这两个标志位，阻止后续的 NAPI 调用）。

而这个 race 就是网卡在 `napi_complete_done()` 的时候，没有正常清除掉 `NAPI_STATE_SCHED` 这个置位，导致 `napi_disable_locked()` 一直卡在死循环中出不来，因为没有人去清理 SCHED 了。

发生这个问题的原因，就是在 `poll_one_napi()` 中置位了 `NAPI_STATE_NPSVC`，这就让上面的 `napi_complete_done()` 直接退出，跳过了清理 SCHED，最终引发错误。

综述一下，发生 race 的调用栈就如 Jakub Kicinski 贴的那样，

```
  [netpoll]                                   [normal NAPI]
                                        napi_poll()
                                          have = netpoll_poll_lock()
                                            rcu_access_pointer(dev->npinfo)
                                              return NULL # no netpoll
                                          __napi_poll()
					    ->poll(->weight)
  poll_napi()
    cmpxchg(->poll_owner, -1, cpu)
      poll_one_napi()
        set_bit(NAPI_STATE_NPSVC, ->state)
                                              napi_complete_done()
                                                if (NAPIF_STATE_NPSVC)
                                                  return false
                                           # exit without clearing SCHED
```

注意这里可以是不同的 CPU，因为 netpoll 非常特殊，它并不是在软中断上下文发生的，而是正常的内核态，例如 printk 的上下文，对于 netconsole 而言它调用 poll 的目的只是在于清理掉发包所分配的资源（DMA 等）。

这里其实还涉及到了 `netpoll_setup()`，换言之在 normal NAPI 执行期间就已经 1. 在 `__napi_poll()` 之后，但是 `napi_complete_done()` 之前执行了 `netpoll_setup()` 2. 再执行了 `poll_napi()`，这个窗口非常非常的小，所以才埋了这么多年都无人发现，而因为 virtio-net 在 poll 的时候要与 VMM 进行通信，这就导致 `napi->poll()` 的耗时要比正常网卡的略高一些，因此增加了触发概率，真的非常巧合了🤯。

# Fixes

修复方面，因为在网卡 NAPI 收包时会执行 `netpoll_poll_lock()`，它首先会通过 `rcu_access_pointer()` 拿到 netpoll 的状态（启用与否），如果启用就会进入一个忙等，

```c
static inline void *netpoll_poll_lock(struct napi_struct *napi)
{
	// ----- snippet -----
	if (dev && rcu_access_pointer(dev->npinfo)) {
		int owner = smp_processor_id();
		while (cmpxchg(&napi->poll_owner, -1, owner) != -1)
			cpu_relax();
		return napi;
	}
	return NULL;
	// ----- snippet -----
}
```

这也是为什么上面 race 会发生，因为按照正常来说，当那边 `netpoll_setup()` 完成后，后续的 NAPI 都应该在上面这里 `cmpxchg(&napi->poll_owner)` 一下，来保证 netpoll 没有占用。

因此修复思路就是让 `rcu_access_pointer()` 更“准确”一点。我们就可以使用 `synchronize_rcu()` 等待包括 normal NAPI 在内的其他所有 CPU 都度过宽限期，即都完成过一次上下文切换，也即我们就保证了只有当没有其他 NAPI 在运行时才完成 `netpoll_setup()`。

```diff
diff --git a/net/core/netpoll.c b/net/core/netpoll.c
index a1da97b5b30b..5f65b62346d4 100644
--- a/net/core/netpoll.c
+++ b/net/core/netpoll.c
@@ -768,6 +768,13 @@ int netpoll_setup(struct netpoll *np)
 	if (err)
 		goto flush;
 	rtnl_unlock();
+
+	/* Make sure all NAPI polls which started before dev->npinfo
+	 * was visible have exited before we start calling NAPI poll.
+	 * NAPI skips locking if dev->npinfo is NULL.
+	 */
+	synchronize_rcu();
+
 	return 0;
 
 flush:
```

其实 `rcu_access_pointer(x)` 相当于一个最小化的 `rcu_read_lock(); READ_ONCE(x); rcu_read_unlock();`，因为在实现上来看 `rcu_read_lock()` 就是关闭了抢占（标准 PREEMPT 模式下，现在貌似改动还挺大的，但语义基本上不怎么会变）来实现宽限期，而单条原子指令 `READ_ONCE` 的执行本身不存在任何被抢占的可能，因此语义上是等价的，这样或许会更加容易理解这个改动。

关于 RCU 的一些事情可以看 [RCU's Requirements](https://www.kernel.org/doc/Documentation/RCU/Design/Requirements/Requirements.html) 这篇介绍。

+++
title = "Asterinas 导览"
date = "2025-03-18"

[taxonomies]
tags = ["entry"]
+++

目前还属于早期阶段，调度器这些都很残缺，不过基本上啥都能跑，只是性能或许不太好。调试环境我还是直接上的 Nix，方便可重复 <https://github.com/plxty/n9/blob/main/shell/asterinas.nix>（。

还有对 ARM64 支持 = 0，感觉有空了可以折腾以下实现下这个架构玩玩。

# Boot

还是经典的 boot，主核心 boot 位于 `ostd/arch/x86/boot/bsp_boot.S`，


1. 执行 `page_table_setup`，设置好页表
2. 入栈 `long_mode_in_low_address` 函数，最后使用 retf 进行 far jump
3. 继续 jmp 到 `long_mode`，此时内核已经映射在了高位地址（ `KERNEL_VMA = 0xffffffff80000000`）
4. 统一的入口执行，在 `__linux64_boot` 阶段（最开始开机的时候）使用的是 `ENTRYTYPE_LINUX_64`，因此会选择 `entry_type_linux` 
5. 最后走到了 `__linux_boot` 阶段，此时就会切换到 Rust 代码了

代码位于 `ostd/src/arch/x86/boot/linux_boot/mod.rs`，


1. 将 EFI 传递的参数（unsafe）存放到全局变量 `EARLY_INFO` 中，其中包括 cmdline、内存布局等信息
2. 调用 `call_ostd_main` 进入早期的初始化阶段
3. 执行 `ostd/src/lib.rs::init` 函数，基本上就是初始化 CPU、串口驱动、内存分配器（frame allocator）、完整的页表、RCU、DMA，注意这里只是初始化一些结构体之类的，也会注册一些基础的 bus 和驱动，但总体来说工作不会很多，也没有拉起其他 CPU；最后就是打开中断
4. 还在 init 函数内，注意这里的 `late_init_on_bsp`，这里就会初始化所有的 AP 核心（BSP 为主核心），最终走到 `boot_all_aps` 通过 IPI 中断通知 AP 进行初始化
5. 执行 `kernel/src/lib.rs`，这里注意到是用了一个 `ostd::main` 的 macro 来实现测试所需要的 main 函数，最终会命名为 `__ostd_main` 然后导出。如果 init 进程退出就会跳出 `while !initproc.status().is_zombie()` 的循环，最后直接 exit_qemu。
6. 回到 main，这里会继续执行更多的初始化，例如驱动设备，还有核心的 scheduler 以及 filesystem；这里 scheduler 存放在全局变量 `SCHEDULER` 中。系统相关的组件也会在 `component::init_all` 中初始化，具体模块会去查找 `Components.toml`
7. 继续往下，开始调用 `ap_init` 初始化其他核心的 idle 线程，并通过 affinity 绑定到其他 CPU，这时候其他 CPU 就会调用 `Thread::yield_now()` 进行调度了，idle 嘛（，如果没有可调度的话似乎是会一直 loop，没有 wfi 之类的？
8. BSP 核心也创建一个线程，这个线程同样是 idle，不过会

看到目前的 scheduler 居然还不支持抢占？纯 RT？emmm 好吧能用。

其他核心的 bootup 应该是 `ostd/src/arch/x86/boot/ap_boot.S` 定义的函数，entry 应该是 `ap_real_mode_boot`，由 IPI 触发。

# Scheduler

插播一个有意思的，<https://github.com/asterinas/asterinas/issues/1471>，

```rust
// FIXME: At this point, we need to prevent the current task from being scheduled on another
// CPU core. However, we currently have no way to ensure this. This is a soundness hole and
// should be fixed. See <https://github.com/asterinas/asterinas/issues/1471> for details.
```

看起来是说在 dequeue 的时候，虽然行为上是线程出队了，但实际上这个时候线程还不能被调度，因为它还指向的是 `current` ，后面的函数可能还会用到它，如果被其他 CPU 调度那么意味着可能会发生 data race，不过现在没有方法去保证这个行为，

> We cannot allow the current task to be scheduled on another CPU core unless the switch_to_task/context_switch is complete

这里的 FIFO scheduler 还是参考的 classic 方式，用的是 per-cpu runqueue，可以在 `FIfoScheduler<T>::new` 中看到，每个 rq 都有两个成员， `current` 和 `queue`。

不过人家已经讨论怎么修复了，<https://github.com/asterinas/asterinas/issues/1633>，有空再看看（。

# Spawn

上面的流程主要是创建内核线程的，那么对于用户线程呢？主要入口是 `spawn_user_process`，随后调用 `create_user_process`，基本资源都在 `process_builder.build()` 中分配完成，包括 VMA 等。

同样也遵循 Linux 那样，Process 为 Tasks（Threads）的集合。目前看来内核中表示为 task，用户中看到的为 thread，有点类似 `task_struct` 的思路？看到 `posix_thread` 的抽象层，那么实际上应该是 Task 为 Asterinas 表示“线程”的名称，而在这之下的 posix_thread 是真正的 thread 封装，会将 Task 的数据转换成 POSIX 接口所需要的内容。

执行到 `process.run()` 之后，内核线程开始工作，会走到 `kernel/src/process/posix_thread/builder.rs::build` 中构建的 thread（里面包括设置好了的 `user_task_entry` ）。

因此新的线程都会在 `user_task_entry` 走一遍，执行 `user_mode.execute(…)`，最后到 `UserContext::execute` 函数，参考 `ostd/src/arch/x86/cpu/mod.rs` 部分，直接用的 sysret 返回到用户态，也是 classic 了（。

# Rust

这个项目最大的特点就是 Rust，它实现了内核态的资源再次隔离，文件夹 `ostd` 内的内容包括架构等，是需要用到 unsafe 的，这部分代码按照 Asterinas 描述最终会跑在可信计算上，例如 Intel SGX 等。其他代码则基本都有 `#![deny(unsafe_code)]` 这样的注解让编译器不允许 unsafe 的出现，包括 `kernel` 以及 `osdk` 。但实际上来说这个还是个宏内核，只是内核驱动与核心（ostd）之间隔离的不错，可以依赖 safe rust 保证驱动的安全。可能缺点就是因为强制 safe，所以所有代码都必须是 Rust 写的，“传染性”很高，毕竟 FFI 是肯定 unsafe 的。。。

以及目前看下来对 POSIX 的兼容还可以，initramfs 都是直接用的 x86 下的 ld.so 等。

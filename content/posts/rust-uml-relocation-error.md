+++
title = "Rust-for-Linux UML 问题"
date = "2025-02-16"

[taxonomies]
tags = ["try-catch"]
+++

发现 UML 目前似乎有点问题，主要是 rust flags 貌似不太正确，导致会出 `[   79.760000] subarch: rust_print: Unknown rela relocation: 9` 这样的错误。

用 readelf 看一下，

```bash
# readelf -W -r ./rust_print.ko | grep GOT
# 偏移量             信息             类型               符号值          符号名称 + 加数
000000000000005c  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
000000000000031d  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
000000000000035c  0000004300000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings5EMERG - 4
0000000000000391  0000004400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings5ALERT - 4
00000000000003c6  0000004500000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4CRIT - 4
00000000000003fb  0000004600000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings3ERR - 4
0000000000000430  0000004700000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings7WARNING - 4
0000000000000465  0000004800000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings6NOTICE - 4
000000000000054e  0000004300000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings5EMERG - 4
0000000000000583  0000004400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings5ALERT - 4
00000000000005b8  0000004500000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4CRIT - 4
00000000000005ed  0000004600000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings3ERR - 4
0000000000000622  0000004700000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings7WARNING - 4
0000000000000657  0000004800000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings6NOTICE - 4
000000000000068c  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
00000000000007a9  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
00000000000008aa  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
000000000000092e  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
000000000000099b  0000002400000009 R_X86_64_GOTPCREL      0000000000000000 _RNvNtNtCsgd4OX6xQzCQ_6kernel5print14format_strings4INFO - 4
```

这里的偏移量基本就是代码段的具体位置，可以直接 gdb disass 看出来，例如 0x31d 这个地址，可以看到

```bash
# (gdb) disass 0x31d

samples/rust/rust_print.rs:
44              pr_info!("Rust printing macros sample (init)\n");
0x000000000000031a <+67>:    48 8b 3d 00 00 00 00    mov    rdi,QWORD PTR [rip+0x0]        # 0x321 <_RNvXCs6nzOMokj4Xc_10rust_printNtB2_9RustPrintNtCsgd4OX6xQzCQ_6kernel6Module4init+74>
```

上面的 `0x31d` 地址，就是 `00 00 00 00` 的这个代码，在 insmod 时就会通过 ELF 表找到对应的符号信息。

本来 `GOTPCREL` 指的是相对与 .got 表的偏移，但是因为内核模块中没有 .got 表（linker 都是内核自己进行处理），因此出现这个值确实是不太对的。

看了下 um 默认用的是 pie，

```makefile
KBUILD_RUSTFLAGS += -Crelocation-model=pie
```

但是 x86 上默认是给的 static。这里用 pie 我猜应该还是因为要最终链接成一个 `linux` 可执行的用户文件，因此用 pie 也是合适的，毕竟也要通过 [ld.so](http://ld.so) 链接系统库，但是对于 modules 来说就不太友好了。

最后发现 code model 也有问题居然，用默认的 kernel 都是不行的，必须指定 large 才行（code model 看了下是为了指定操作数的宽度的，如果是 large 则会选择使用更宽的 relocation 操作数）。

在生成代码的时候，如果 code model 是 small，那么给它生成的地址宽度一般就只有 32bit 大小，这种适合相对位置的重定向。但是内核链接的时候，使用的是符号的绝对地址（这样或许能够提升一点点的性能），而在 x86 64 下，绝对地址的大小可能会轻松就超过 32bit，这就导致了内核的校验出现了问题。

```diff
diff --git a/arch/um/Makefile b/arch/um/Makefile

index 00b63bac5eff..248e94133855 100644
--- a/arch/um/Makefile
+++ b/arch/um/Makefile
@@ -63,7 +63,8 @@ KBUILD_CFLAGS += $(CFLAGS) $(CFLAGS-y) -D__arch_um__ \
 	-Din6addr_loopback=kernel_in6addr_loopback \
 	-Din6addr_any=kernel_in6addr_any -Dstrrchr=kernel_strrchr
 
-KBUILD_RUSTFLAGS += -Crelocation-model=pie
+KBUILD_RUSTFLAGS_KERNEL += -Crelocation-model=pie
+KBUILD_RUSTFLAGS_MODULE += -Ccode-model=large
 
 KBUILD_AFLAGS += $(ARCH_INCLUDE)
```

目前暂时这么解决了，不知道是否需要汇报给一下社区？或者只是因为我 openSUSE 的问题吗。。

理论上 Rust 的后端是 LLVM，看了下

```cpp
bool X86DAGToDAGISel::selectMOV64Imm32(SDValue N, SDValue &Imm) {
  // Cannot use 32 bit constants to reference objects in kernel/large code
  // model.
  if (TM.getCodeModel() == CodeModel::Kernel ||
      TM.getCodeModel() == CodeModel::Large)
    return false;
```

LLVM 应该是会正确处理这个问题才对啊，kernel 的时候就不会允许使用 32 位的立即数了。

```bash
# MUST have currently, to forcebily enable nightly features...
export RUSTC_BOOTSTRAP=1

RUST_MODFILE=samples/rust/rust_print rustc --edition=2021 -Zbinary_dep_depinfo=y -Dunsafe_op_in_unsafe_fn -Dnon_ascii_idents -Wrust_2018_idioms -Wunreachable_pub -Wmissing_docs -Wrustdoc::missing_crate_level_docs -Wclippy::all -Wclippy::mut_mut -Wclippy::needless_bitwise_bool -Wclippy::needless_continue -Wclippy::no_mangle_with_rust_abi -Wclippy::dbg_macro -Cpanic=abort -Cembed-bitcode=n -Clto=n -Cforce-unwind-tables=n -Ccodegen-units=1 -Csymbol-mangling-version=v0 -Crelocation-model=static -Zfunction-sections=n -Wclippy::float_arithmetic --target=./scripts/target.json -Ctarget-feature=-sse,-sse2,-sse3,-ssse3,-sse4.1,-sse4.2,-avx,-avx2 -Copt-level=s -Cdebug-assertions=n -Coverflow-checks=y -Cforce-frame-pointers=y -Cdebuginfo=2  --cfg MODULE \
	-Ccode-model=kernel \
	@./include/generated/rustc_cfg -Zallow-features=new_uninit -Zcrate-attr=no_std -Zcrate-attr='feature(new_uninit)' -Zunstable-options --extern force:alloc --extern kernel --crate-type rlib -L ./rust/ --crate-name rust_print --sysroot=/dev/null --out-dir samples/rust/ --emit=dep-info=samples/rust/.rust_print.o.d --emit=obj=samples/rust/rust_print.o \
	samples/rust/rust_print.rs
```

稍微测试一下。。

看了下 GCC 文档，明白了。。

> `-mcmodel=kernel`
>
> Generate code for the kernel code model. The kernel runs in the negative 2 GB of the address space. This model has to be used for Linux kernel code.

所以 UML 必须使用 large，并且 `arch/um/Makefile` 其实都已经写了，只是 Rust 这边并没有同步到的样子。。感觉可以提个 issue 或者是简单的 patch？

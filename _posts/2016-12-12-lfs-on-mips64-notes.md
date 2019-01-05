---
layout: post
title:  "LFS on MIPS64 笔记"
date:   2016-12-12
categories: mips lfs
---

**Update:** 修订于 2018-12-23 / 2019-1-5，第二次 AOSC OS MIPS Port Bootstrap。

写这篇文章的时候我对 LFS 和 MIPS 都一知半解。对我新一次构建的笔记，参见 [LFS on MIPS64 全过程笔录](https://lmy441900.github.io/mips/lfs/2018/12/23/lfs-on-mips64-process.html)。

---

最近在做 AOSC OS 的 MIPS64el 移植相关尝试，目标平台是 MIPS64r2 通用，采用 N64 ABI。因为不想在交叉编译的 Stage 1 上花费时间，加上我手上这台 MIPS 机器性能足够好（一台[龙芯 3A2000C](http://loongson.cn/product/cpu/3/Loongson3A2000.html)，8 GB 内存），就想直接基于龙芯开源社区的 Loongnix Linux 进行 LFS。以下是我踩过一些坑以后的总结。

首先是过程和[官方 LFS](http://www.linuxfromscratch.org/lfs/view/stable-systemd/index.html) 大致上并无差异，我在参考 [CLFS (Pure 64)](http://www.clfs.org/view/CLFS-3.0.0-SYSTEMD/mips64-64/index.html) 和 AOSC OS 基本参数的基础上大致遵守了 LFS 的构建规范，毕竟这也确实是一个 LFS 过程。一开始的几次失败让我感觉到玄学特多，但是思路清晰了之后发现其实不过是一些 MIPS 平台特有的特性在影响构建过程，以及 LFS 本身的坑。

## MIPS 特有属性

MIPS 平台和 x86 平台的一个比较大的差异就在于 [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)。在 MIPS 平台上同时存在[三种 ABI](https://www.linux-mips.org/wiki/WhatsWrongWithO32N32N64)：

- o32（`-mabi=32`）
- n32（`-mabi=n32`）
- n64（`-mabi=64`）

另外其实还存在 o64、EABI、NUBI 和别的奇葩 ABI，实际上我们发现 n32 也是比较冷门的 ABI，至少几个主流的移植到 MIPS 的发行版没有选择这个 ABI。如果单纯按照 LFS 的构建过程编译工具链，出来的东西将是 n32 的（如果你的编译器支持并默认是这样，至少 Loongnix 是）。所以在构建 Temporary System 的时候，要注意在 GCC Pass 1 和 Pass 2 的时候都加上 `configure` 参数：

```bash
# ...
--with-abi=64          \ # 设定默认 ABI 为 n64
--with-arch=mips64r2   \ # 设定默认 march 为 mips64 release 2
--with-tune=loognson3a \ # 设定默认 mtune 为 loongson3a（因为构建机器是这个）
# ...
```

这是 AOSC OS 的例子，具体参数可以更改。实际上我为了让整个工具链都调校到这个配置（`gcc` 的默认 march 是 mips3），在一开始的 Binutils 构建我调整了 `CC` 和 `CXX` 变量，使得：

```bash
export BUILD64="-mabi=64 -march=mips64r2 -mtune=loongson3a"
export LFS_BLD=mips64el-redhat-linux   # Loongnix Fedora 21 Remix
export LFS_TGT=mips64el-aosc-linux-gnu
CC="$LFS_BLD-gcc $BUILD64" CXX="$LFS_BLD-g++ $BUILD64" \
../configure # ...
```

（没错确实应该放到 `$CC` 和 `$CXX` 里，具体请参阅每个变量的含义~~，也是为了方便操作~~）

GCC 亦如此。对于 Loongnix 来说，因为它的 GCC 已经是默认这套参数了，所以实际上这个步骤不必要。如果是其它 Host，那么可能还是需要这么做。

## LFS 的坑

除了 MIPS 平台带来的一些麻烦，我发现 LFS 本身也存在问题，具体地说就是从 binutils Pass 2 开始 LFS 就出现了在 `configure` 前面使用 `CC=$LFS_TGT-gcc`、`AR=$LFS_TGT-ar` 等变量强行用 Pass 1 的 GCC 编译。**这是 ~~错误~~ _不完整_ 的**，并且不难发现 `make install` 以后在 `/tools` 下多出一个 `mips64el-unknown-linux-gnu/`。这个 Triplet 是 `config.guess` 的结果——换句话说，这个构建参数是不正确的，应该还需要让 `configure` 配置使用 GCC Pass 1 来构建 Binutils Pass 2 以及 GCC Pass 2：

```bash
CC=$LFS_TGT-gcc                \
AR=$LFS_TGT-ar                 \
RANLIB=$LFS_TGT-ranlib         \
../configure                   \
    --build=$LFS_TGT           \ # <-
    --prefix=/tools            \
    --disable-nls              \
    --disable-werror           \
    --with-lib-path=/tools/lib \
    --with-sysroot
```

但是为什么还是要 Override `CC` 呢？这是因为在 LFS 工具链中还不存在 `gcc` 这一链接，而 `configure` 找 GCC 的时候，第一个就找到了 `gcc`，恰好这个 GCC 并不是我们工具链中的那个，是宿主系统的那个。LFS 里还覆盖了 `AR` 和 `RANLIB`，我用 `which` 看了一下这两个都指向刚刚构建的工具链里的 Binutils，应该（理论上）是不用覆盖的吧。

在整个构建过程中，**最好预先设定 `--build` 和 `--host`（当然还有 `--target`，如果有的话）**。否则 Triplet 就会被 Guess，而如果你的 Triplet 不是 `config.guess` 给出的默认值，那就搞错了。

## 别的问题

1. 因为 AOSC OS 不采用 Multilib 方式构建 32/64 位共存系统（AOSC OS 使用 32Subsystem，“32 位子系统”，因此宿主系统是 Pure 64 的），所以~~根据 CLFS 在构建 Binutils 的时候应该加上 `--enable-64-bit-bfd` 来开启 64 位支持。（实际上 LFS 也是纯 64 的，莫非是自动开启的？这样就没有加这个开关的必要了。）~~
    - 6.16. Binutils-2.31.1: _"May not be needed on 64-bit systems, but does no harm."_
2. **GNU Gold Linker 没法用**，所以~~不要打开 `--enable-gold`。虽然平台支持，但是 `ld.gold` 相对于 `ld.bfd` 有很多特性及开关缺失，导致诸如 `configure` 这样的构建系统不认（另外也认不出 `ld` 的版本号，就会报错，特别是 `glibc`）。听说在 x86 平台上没有这些问题，不是很懂。~~
    - ~~然而实际情况是我们在 `mips64el` 上仍然打开了 Gold Linker，问题不算太大（打了补丁）……先看着办吧。~~
    - `--enable-gold` 可以，`--enable-default-gold` 就不清楚了，过了两年了不知道 gold 在 MIPS 上支持怎么样了。

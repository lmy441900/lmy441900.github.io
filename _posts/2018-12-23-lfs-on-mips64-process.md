---
layout: post
title:  "LFS on MIPS64 全过程笔录（持续更新）"
date:   2018-12-23
categories: mips lfs
---

> 明年年初，社区合作的 MIPS Port 即将（第二次）正式开机，我将继续扮演架构维护者，我会用维护者技术手段努力创造一个能用的形象，文体两开花，弘扬 MIPS 文化，希望大家多多关注

## 编译平台

> Junde Yhi, [22.12.18 22:50]
> [In reply to Neo_Chen (neo_chen@NeoVAX)]
> Host 是由江苏龙芯梦兰公司提供的 Fedora 28 (Loongson ver.) 一套
>
> Junde Yhi, [22.12.18 22:51]
> 在此我们谨对 sunhaiyong 先生表示我们一贯的崇高敬意
>
> Junde Yhi, [22.12.18 22:51]
> 没有 sunhaiyong 就没有 aosc [os mips port]

## 目标平台

1. AOSC OS MIPS64EL "Care"[^1] Port
  - `mips64el-aosc-linux-gnuabin64`
  - (Legacy) **MIPS-III** ISA
2. AOSC OS MIPS64EL "Modern" Port
  - `mipsisa64r2el-aosc-linux-gnuabin64`
  - **MIPS64 revision 2** ISA

LFS 过程基本遵循标准 LFS 教程，因此我只按 LFS 步骤记下要注意或者要添加的事项。

## 4.4. Setting Up the Environment

- 注意调整 `$LFS_TGT` 到目标平台 Triplet
- 可以考虑添加一些常用变量：
  - `$LFS_BLD` 当前平台 Triplet，因为 `config.guess` 给的 `*-unknown-*` 还是丑了点
  - `$MAKEFLAGS` 放 `-j$(( $(nproc) + 1 ))`（执行 `nproc` 得到当前 CPU 数，加一）让 `make` 多线程编译，而不需要每次都打 `-jN`

## 5.4. Binutils-2.31.1 - Pass 1

- 注意添加 `--build=$LFS_BLD`（实际上可能没有意义）
- 按照 AOSC OS 文件系统结构定义，LFS 给出的在 x86_64 上的 `lib64 -> lib` 链接也需要做好

## 5.5. GCC-8.2.0 - Pass 1

- 在修改 GCC 默认链接器位置的时候（_The following command will change the location of GCC's default dynamic linker to use the one installed in /tools_），注意切换 `i386` 到 `mips`
  - 并且 `mips` 下没有 `linux64.h`，所以可以免去这一替换
- LFS 对 `x86_64` 修改了默认的 64 位链接库目录到 `lib`（_Finally, on x86\_64 hosts, set the default directory name for 64-bit libraries to "lib"_）；对 MIPS，我尚不清楚位于 `gcc/config/mips/t-linux64` 的类似修改是否有意义，因为这里面的 `lib{,32,64}` 定义是对 Multiarch 的，在没有 Multiarch 的情况下应该不起效，所以我没有修改这一项。
- 在 `configure` 处：
  - 添加 `--build=$LFS_BLD`
  - 对 "Care" Port，添加 `--with-abi=64 --with-arch=mips3 --with-tune=loongson2f`
  - 对 "Modern" Port，添加 `--with-abi=64 --with-arch=mips64r2 --with-tune=loongson3a`[^2]
    - 另添加 `--without-madd4` 绕过龙芯三号对 MIPS 标准的不兼容实现（unfused -> fused），会有性能损失，但是是兼容性的代价
  - 添加 `--disable-multilib`，目标构建是纯 64 位系统

## 5.7. Glibc-2.28

- 在 `configure` 处：
  - **这里 LFS 指定了 `--build=$(../scripts/config.guess)`，注意我们的 Triplet 和猜的不同，所以需要替换成 `--build=$LFS_BLD`**

## 5.8. Libstdc++ from GCC-8.2.0

- 添加 `--build=$LFS_BLD`

## 5.9. Binutils-2.31.1 - Pass 2

- 在 `configure` **后**：
  - 添加 `--build=$LFS_TGT`
    - 注意这里要开始使用 Pass 1 GCC 了，是 `TGT`
    - 并且我们的 Triplet 和猜的不同，所以这一项是必要的

## 5.10. GCC-8.2.0 - Pass 2

- 在 `configure` **后**：
  - 添加 `--build=$LFS_TGT`
    - 同上，这里要开始使用 Pass 1 GCC 了，是 `TGT`
    - 并且我们的 Triplet 和猜的不同，所以这一项是必要的
- 其余步骤参见 Pass 1

## 5.29. Perl-5.28.0

- Perl 的非标准 `Configure` 在这个阶段会两次莫名其妙地进入 Bash 的交互命令行，我不太清楚是怎么回事。

## The Rest

- 为了好看，记得 `--build=$LFS_TGT`

## 6.6. Creating Essential Files and Symlinks

- 在 _6.7. Linux-4.18.5 API Headers_ `make mrproper` 的时候还需要 `/bin/awk -> /bin/gawk -> /tools/bin/gawk` 这个链接，加上

## 6.9. Glibc-2.28

- 对 _Determine the GCC include directory_ 一操作，由于 LFS 中指定了是 x86 架构的操作，这里需要手动 `export`
- 我们不 LSB，两个 Symlink 可以不用做
  - 否则还是需要做的，目标库文件名是 `ld.so.1`

## 6.17. GMP-6.1.2

- 由于在 MIPS 上 GMP 支持三种 ABI，在 `configure` 前手动指定 `ABI=64`（或者 `export ABI=64`）
- GMP 带的 `configure` 对 Triplet 的识别不是很标准，把我们 `mipsisa64r2el` 的 CPU 判断成了 32 位的。这里要给 `configure:4664` 做个改动：
  - `mips64*-*-* -> mips*64*-*-*`
  - 如果 Triplet 不是那么奇葩，这里是不需要动的

## Notes

[^1]: [Forwarded from imi415] aosc os，关爱您、您的开发板、您的谜之处理器和您的史前遗产
[^2]: 对龙芯三号的 `LL` / `SC` Bug，最好是通过 Binutils 补丁的方式让 `gas` 修复问题，而不是在 GCC 这里 `--without-llsc`，这样性能下降会比较厉害，不推荐。

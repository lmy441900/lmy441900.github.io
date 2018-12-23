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
- 暂时没有找到方法修改 MIPS 上默认的 64 位链接库目录到 `lib`（_Finally, on x86\_64 hosts, set the default directory name for 64-bit libraries to "lib"_）
- 在 `configure` 处：
  - 添加 `--build=$LFS_BLD`
  - 对 "Care" Port，添加 `--with-abi=64 --with-arch=mips3 --with-tune=loongson2f`
  - 对 "Modern" Port，添加 `--with-abi=64 --with-arch=mips64r2 --with-tune=loongson3a`

## 5.7. Glibc-2.28

- 在 `configure` 处：
  - **这里 LFS 指定了 `--build=$(../scripts/config.guess)`，注意我们的 Triplet 和猜的不同，所以需要替换成 `--build=$LFS_BLD`**

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

## Notes

[^1]: [Forwarded from imi415] aosc os，关爱您、您的开发板、您的谜之处理器和您的史前遗产

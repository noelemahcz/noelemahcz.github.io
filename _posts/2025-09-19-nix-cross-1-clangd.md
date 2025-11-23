---
title: "Nix 交叉编译踩坑记（一）之 clangd 与 --query-driver"
categories: [DevEnv, Nix]
tags: [Nix, Cross Compile]
---

## 起因

故事要从我打算开发一个在 **Windows** 和 **WSL2** 之间通过 DragDrop 来传输文件的小工具开始讲起。

实现 DragDrop 自然需要有一个窗口作为拖放的源或目标，这就涉及到了 Win32 API，意味着这个工具必须是一个原生 Windows 可执行文件，而众所周知 Nix 这套工具自身是不支持运行在 Windows 系统上的，这不可避免要引入交叉编译环境。

好在 Nix 对于交叉编译是一类支持，Nixpkgs 的 `default.nix (top-level/impure.nix)` 实际上是接受 `localSystem` 和 `crossSystem` 作为参数的函数，这两个参数分别对应交叉编译技术中的 `Build` 和 `Host` 概念，Nix 根据这两个参数配置 `pkgs.stdenv`，进而构建出这个 `pkgs` 下的绝大多数软件包。

## Autotools 背大锅

在进入 Nix 世界之前，我觉得有必要先简单解释下 `Build`, `Host`, `Target` 这三个交叉编译概念（术语）。

<!-- prettier-ignore-start -->
> 由于 `Target` 概念的存在导致交叉编译技术理解起来有些困难，甚至 [Nixpkgs Reference](https://nixos.org/manual/nixpkgs/stable/#ssec-cross-platform-parameters) 也称之为“历史错误”，但如今的软件世界已经充满了 `Target` 概念，Nixpkgs 也只能选择支持它。
{: .prompt-info }
<!-- prettier-ignore-end -->

正确理解这三个概念的方式之一是通过“视角锁定”技巧，我们假设当前处于一种涉及三个不同平台的交叉编译场景：

- 一台 **x86-64 Linux** 老旧主机，它有一个满足条件的专用 GCC 。
- 一台 **AArch64 Darwin** 高性能 Mac Pro 主机，它没有任何编译器。
- 一台 **ARMv6l Linux** 树莓派。

此场景的目标是使用老旧 Linux 主机上的 GCC 为高性能 Mac Pro 主机编译一个新的 GCC，这个新 GCC 运行在 Mac Pro 上但为 RaspberryPi 生成机器码，这是一个典型的 [**Canadian Cross**](https://en.wikipedia.org/wiki/Cross_compiler#Canadian_Cross) 场景。

### GCC-old

我们首先将“视角”锁定在位于老旧 Linux 主机的 GCC (GCC-old) 上：

- `Build`：由于我们已经拥有了完整的 GCC-old 可执行文件，因此可以不必太过关心它的 `Build` 平台是什么，它可能是通过本机上的另一个 GCC 原生编译而来，也可能经由其它交叉或非交叉平台的主机编译后复制而来，总之不那么重要。
- `Host`：就是 GCC-old 的运行平台，当前场景下就是老旧主机所使用的平台：**x86-64 Linux**，就这么简单。
- `Target`：前面提到过 `Target` 平台的存在可能是一个历史错误，这是由于 GCC 是单目标编译器，也就是说任意一个 GCC 要么为 x86-64 Linux 生成机器码，要么为其它某个平台生成机器码，因此 GCC 与其 `Target` 具有紧密的绑定。在当前场景下，GCC-old 用于编译出一个新的、给 Mac Pro 使用的 GCC-new，所以它的 `Target` 平台是 **AArch64 Darwin** 。

阶段性总结一下，GCC-old 的 `Build`, `Host`, `Target` 平台分别为 (**Whatever**, **x86-64 Linux**, **AArch64 Darwin**) 。

### GCC-new

然后我们将视角切换并锁定在是被 GCC-old 编译出的 GCC-new 上：

- `Build`：因为 GCC-new 从 GCC-old 编译而来，所以 GCC-new 的 `Build` 平台就是 GCC-old 的 `Host`（运行）平台，即 **x86-64 Linux** 。
- `Host`：GCC-new 的运行平台，同时也是 GCC-old 的 `Target` 平台：**AArch64 Darwin**，依旧简单。
- `Target`：因为 GCC-new 为 RaspberryPi 生成代码，所以 `Target` 是 **ARMv6l Linux** 。

<!-- prettier-ignore-start -->
> 任何 GCC 的 `Target` 平台都是在 configure 阶段通过 `--target` 参数指定的，就当前场景具体而言：
- GCC-new 经过 `./configure --target=armv6l-unknown-linux-gnueabihf` 命令配置后使用 GCC-old 编译获得。
- GCC-old 则通过 `./configure --target=x86_64-unknown-linux-gnu` 进行配置，只不过我们已装有 GCC-old 这个成品，并不需要亲自编译它。
{: .prompt-info }
<!-- prettier-ignore-end -->

## The Nix Way: `stdenv`
通常我们会使用 `let pkgs = import <nixpkgs> {} in ...;` 以省略参数的形式导入 Nixpkgs，Nix 会将 `localSystem` 设定为当前运行 Nix 的平台，且 `crossSystem` 默认等于 `localSystem`，这意味着 `pkgs` 代表了一个**非交叉编译**的原生软件包集合，`pkgs.stdenv.buildPlatform` 和 `pkgs.stdenv.HostPlatform` 都为当前平台（比如 `x86_64-unknown-linux-gnu`）。

在我的 NixOS 上：

```bash
$ nix eval --impure --expr 'let pkgs = import <nixpkgs> {}; in with pkgs.stdenv; [buildPlatform.config hostPlatform.config]'
[ "x86_64-unknown-linux-gnu" "x86_64-unknown-linux-gnu" ]
```

<!-- prettier-ignore-start -->
> Although the existence of a “target platform” is arguably a historical mistake, it is a common one: examples of tools that suffer from it are GCC, Binutils, GHC and Autoconf. Nixpkgs tries to avoid sharing in the mistake where possible. Still, because the concept of a target platform is so ingrained, it is best to support it as is.
{: .prompt-info }
<!-- prettier-ignore-end -->

如上所述，`nixpkgs` 深受交叉编译平台三元组的影响，使得有关于 `pkgs` / `stdenv` / `depsXXXYYY` 的概念变得复杂，以致于我每隔一段时间再次接触这些交叉编译细节都会被搞晕一次，所幸理清思绪所需要的时间也一次次变短，终于在今天，我决定写点什么记录下来，一是真的需要写下来，二是我可能终于有能力写点什么（哪怕只是给自己看的杂乱的笔记）。

首先 `nixpkgs` 定义了大量的包集，也就是 `pkgs???`，每一个 `pkgs???` 都是通过平台三元组（交叉）编译而来的软件包的集合。

以默认参数 `import` 得到的包集的 **Build** **Host** **Target** 平台都为当前平台：

```bash
[I] noelemahcz@nixos ~> nix eval nixpkgs#legacyPackages.x86_64-linux --apply 'pkgs: with pkgs; [buildPlatform.config hostPlatform.config tar
getPlatform.config]'
[ "x86_64-unknown-linux-gnu" "x86_64-unknown-linux-gnu" "x86_64-unknown-linux-gnu" ]
```

这实际上并非交叉编译包集，在这个 "default" pkgs 下，我们还有 pkgsCross 属性集下的一系列*真*交叉编译包集：

```bash
[I] noelemahcz@nixos ~> nix eval nixpkgs#pkgsCross --apply builtins.attrNames
[ "aarch64-android" "aarch64-android-prebuilt" "aarch64-darwin" "aarch64-embedded" "aarch64-freebsd" "aarch64-multiplatform" "aarch64-multiplatform-musl" "aarch64be-embedded" "arm-embedded" "arm-embedded-nano" "armhf-embedded" "armv7a-android-prebuilt" "armv7l-hf-multiplatform" "avr" "ben-nanonote" "bluefield2" "fuloongminipc" "ghcjs" "gnu32" "gnu64" "gnu64_simplekernel" "i686-embedded" "iphone64" "iphone64-simulator" "loongarch64-linux" "loongarch64-linux-embedded" "m68k" "microblaze-embedded" "mingw32" "mingwW64" "mips-embedded" "mips-linux-gnu" "mips64-embedded" "mips64-linux-gnuabi64" "mips64-linux-gnuabin32" "mips64el-linux-gnuabi64" "mips64el-linux-gnuabin32" "mipsel-linux-gnu" "mmix" "msp430" "musl-power" "musl32" "musl64" "muslpi" "or1k" "pogoplug4" "powernv" "ppc-embedded" "ppc64" "ppc64-elfv1" "ppc64-elfv2" "ppc64-musl" "ppcle-embedded" "raspberryPi" "remarkable1" "remarkable2" "riscv32" "riscv32-embedded" "riscv64" "riscv64-embedded" "riscv64-musl" "rx-embedded" "s390" "s390x" "sheevaplug" "ucrt64" "ucrtAarch64" "vc4" "wasi32" "wasm32-unknown-none" "x86_64-darwin" "x86_64-embedded" "x86_64-freebsd" "x86_64-netbsd" "x86_64-netbsd-llvm" "x86_64-openbsd" "x86_64-unknown-redox" ]
```

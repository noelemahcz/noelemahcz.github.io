---
title: "Nix Cross Dependencies"
categories: [Environment, Nix]
tags: [Nix]
---

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

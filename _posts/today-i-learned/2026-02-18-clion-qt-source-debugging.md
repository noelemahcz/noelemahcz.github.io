---
title: "[TIL] CLion 配置 Qt 源码调试（小坑不算坑）"
categories: [Environment, C++]
tags: [TIL, C++, Qt, CLion]
---

## 一、前言

很久没有用 CLion 了，之前一直在用 VSCode 和 Neovim 进行 Windows 和 Linux 跨平台开发，可定制程度比较高，但去年觉得调试方面还是 CLion 要方便些，遂将 CLion 也加入开发环境大礼包中。

最近几天在写 Qt Meta-Object 相关的文章，需要深入到 Qt 源码中，遂配置 Qt 源码级调试环境。

---

## 二、安装源码和调试符号

这一步没什么可说的，获取 Qt 预编译二进制的主要方式就是 Official Online Installer，安装时同时勾选 "Sources" 和 "Qt Debug Information Files" 项即可，Installer 会将源码安装到如 "C:\Qt\6.10.2\Src" 的位置，调试符号则会和 Qt Binaries 放在一起，如 "C:\Qt\6.10.2\msvc2022_64\bin"。

![Qt Online Installer](/assets/images/posts/today-i-learned/2026-02-18-clion-qt-source-debugging/qt_online_installer.png)

---

## 三、配置 CLion LLDB

默认情况下 CLion 是没法通过 F7 步入进 Qt 源码中的（比如 "qobject.cpp"），而是会跳进反汇编中，这是因为 Qt 官方编译时的源代码路径与 Installer 安装的位置不一致。

我们可以通过 llvm-pdbutil 来查看调试符号中记录的源代码位置：

```
PS C:\Qt\6.10.2\msvc2022_64\bin> llvm-pdbutil dump -modules Qt6Cored.pdb | Select-String 'qobject.cpp'

Mod 0078 | `C:\Users\qt\work\qt\qtbase_build\src\corelib\CMakeFiles\Core.dir\Debug\kernel\qobject.cpp.obj`:
Obj: `C:\Users\qt\work\qt\qtbase_build\src\corelib\CMakeFiles\Core.dir\Debug\kernel\qobject.cpp.obj`:
```

然后我们在项目根目录建立一个 ".lldbinit" 文件，指示 LLDB 建立 Qt 源码路径映射：

```
settings set target.source-map "C:/Users/qt/work/qt" "C:/Qt/6.10.2/Src"
```

这样就可以直接通过 F7 步入进 Qt 的函数中了。

---

## 四、小坑 —— Variables View 为空

经过上述步骤后，我发现 CLion Debug 工具栏中 Variables View 并不显示 Qt 函数的参数和局部变量的实时信息，起初我还以为 Qt 官方提供的符号是经过 Stripped 的，但我印象里并不会，之前用 VSCode 和 LLDB/GDB 都能正常显示。

于是我使用 LLDB 命令行 `print` 命令查看参数信息可以正常显示：

```
(lldb) p sender
(Counter *) $0 = 0x00000039646ff798 {...}
(lldb) p receiver
(Counter *) $1 = 0x00000039646ff798 {...}
(lldb) p signal
(const char *) $2 = 0x00007ff6aa94cb60 "2valueChanged()"
(lldb) p method
(const char *) $3 = 0x00007ff6aa94cb10 "1setValue()"
(lldb) p type
(Qt::ConnectionType) $4 = AutoConnection
```

并且使用 llvm-pdbutil 查看 PDB 信息也显示未经裁剪：

```
PS C:\Qt\6.10.2\msvc2022_64\bin> llvm-pdbutil dump --summary Qt6Cored.pdb

                          Summary
============================================================
  Block Size: 4096
  Number of blocks: 17287
  Number of streams: 739
  Signature: 1768846930
  Age: 1
  GUID: {A2C39570-A0A4-4E4C-886C-76704AC26989}
  Features: 0x1
  Has Debug Info: true
  Has Types: true
  Has IDs: true
  Has Globals: true
  Has Publics: true
  Is incrementally linked: true
  Has conflicting types: false
  Is stripped: false
```

这说明 PDB 文件是没有问题的，我觉得大概率是 CLion 和 LLDB 后端的交互问题，或者是 CLion 的设置问题。

仔细翻了一遍 CLion 的 Debug 相关设置，发现了一项名为 "Hide out-of-scope variables" 的选项默认是勾选的，取消勾选后果然就正常显示了：

![CLion Debug Settings](/assets/images/posts/today-i-learned/2026-02-18-clion-qt-source-debugging/clion_debug_settings.png)

---

## 五、问题解决

至此，可以用 CLion 开心调试 Qt 源码了，效果如下：

![CLion Variables View](/assets/images/posts/today-i-learned/2026-02-18-clion-qt-source-debugging/clion_variables_view.png)

> 不确定这是 CLion 的 BUG 还是就是这样设计的，按理说进入到 Qt 函数的作用域后不应该判定其参数和局部变量 "out-of-scope"。
{: .prompt-info }

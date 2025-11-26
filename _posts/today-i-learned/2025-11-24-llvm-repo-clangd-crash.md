---
title: "[TIL] LLVM 开发环境 clangd crash 问题排查"
categories: [Environment, C++]
tags: [TIL, C++, LLVM, clangd]
---

## 起因

为了分析 LLVM 某个特性的旧版本实现，我将 llvm repo 切换到了 `v18.1.8` 分支，并在独立的 build 目录中重新运行了 CMake，用 nvim 打开相关源文件，clangd 开始工作，建立了 4000+ 个 TU 的索引，但在最后的 70+ 个 TU 上索引失败，直接 crash 掉。

我查看 `lsp.log` 文件定位到了关键行：

```
[13:28:43.256] Indexed /home/noelemahcz/llvm-project/clang/lib/Sema/SemaExceptionSpec.cpp (16341 symbols, 95605 refs, 591 files)
[13:28:43.256] Failed to compile /home/noelemahcz/llvm-project/clang/lib/Sema/SemaExceptionSpec.cpp, index may be incomplete
[Error - 13:28:43 PM] Server process exited with signal SIGSEGV.
```

可以看到错误与 `SemaExceptionSpec.cpp` 文件有关，于是我使用 `clangd --check=clang/lib/Sema/SemaExceptionSpec.cpp --log=verbose` 命令启用详细输出单独检查这个源文件：

```
I[15:19:00.229] Indexing headers...
V[15:19:00.639] indexed preamble AST for /home/noelemahcz/llvm-project/clang/lib/Sema/SemaExceptionSpec.cpp version null:
  symbol slab: 55866 symbols, 16193460 bytes
  ref slab: 0 symbols, 0 refs, 128 bytes
  relations slab: 1553 relations, 34840 bytes
V[15:19:00.774] Build dynamic index for header symbols with estimated memory usage of 28151124 bytes
E[15:19:00.788] [pp_file_not_found] Line 13: in included file: 'clang/Sema/AttrParsedAttrList.inc' file not found
I[15:19:00.788] Building AST...
E[15:19:00.946] [unknown_typename] Line 82: unknown type name 'ExprResult'
E[15:19:00.946] [init_conversion_failed] Line 88: cannot initialize return object of type 'int' with an lvalue of type 'Expr *'
E[15:19:00.946] [unknown_typename] Line 92: unknown type name 'ExprResult'
E[15:19:00.946] [init_conversion_failed] Line 102: cannot initialize return object of type 'int' with an rvalue of type 'ConstantExpr *'
E[15:19:00.946] [no_member] Line 613: no member named 'insert' in 'llvm::SmallPtrSet<clang::CanQual<clang::Type>, 8>'
E[15:19:00.946] [no_member] Line 620: no member named 'size' in 'llvm::SmallPtrSet<clang::CanQual<clang::Type>, 8>'
E[15:19:00.946] [member_function_call_bad_type] Line 1100: cannot initialize object parameter of type 'const clang::Expr' with an expression of type 'const CXXDynamicCastExpr'
E[15:19:00.946] [member_function_call_bad_type] Line 1103: cannot initialize object parameter of type 'const clang::ExplicitCastExpr' with an expression of type 'const CXXDynamicCastExpr'
E[15:19:00.946] [ovl_no_viable_member_function_in_call] Line 1106: no matching member function for call to 'getSubExpr'
E[15:19:00.946] [member_function_call_bad_type] Line 1109: cannot initialize object parameter of type 'const clang::CastExpr' with an expression of type 'const CXXDynamicCastExpr'
E[15:19:00.946] [member_function_call_bad_type] Line 1150: cannot initialize object parameter of type 'const clang::Expr' with an expression of type 'const clang::CXXDynamicCastExpr'
E[15:19:00.946] [ovl_no_viable_function_in_call] Line 1155: no matching function for call to 'canSubStmtsThrow'
E[15:19:00.946] [member_function_call_bad_type] Line 1214: cannot initialize object parameter of type 'const clang::Expr' with an expression of type 'const clang::CXXNewExpr'
E[15:19:00.946] [init_conversion_failed] Line 1217: cannot initialize a parameter of type 'const Expr *' with an lvalue of type 'const clang::CXXNewExpr *'
E[15:19:00.946] [ovl_no_viable_function_in_call] Line 1220: no matching function for call to 'canSubStmtsThrow'
E[15:19:00.946] [no_member_suggest] Line 1317: no member named 'OMPArraySectionExprClass' in 'clang::Expr'; did you mean 'ArraySectionExprClass'?
E[15:19:00.946] [no_member_suggest] Line 1382: no member named 'TypoExprClass' in 'clang::Expr'; did you mean 'AsTypeExprClass'?
E[15:19:00.946] [duplicate_case] Line 1382: duplicate case value 'AsTypeExprClass'
```

很幸运，一次性定位到了错误发生的根本原因，即找不到 `clang/Sema/AttrParsedAttrList.inc` 文件，由此引发了后续一系列的类型未定义或不匹配错误，最终导致 clangd emit 大量 errors 进而 crash 掉。

---

## 解决方案

LLVM 项目中存在大量 **TableGen** (`.td`) 文件，这是一种 DSL 格式，由 `clang-tblgen` 工具在项目构建时动态生成 `.inc` 文件。

`AttrParsedAttrList.inc` 文件定义了 clang 支持的所有 Attributes，作为源码的一部分，如果缺失肯定会导致 clangd 解析出错，因此仅运行 CMake configure 是不够的，但完整构建整个 LLVM 项目对于我临时查看旧版本实现的需求来说成本又有些过高了。

于是我想到查看 ninja 的 targets，看看有没有单独的 TableGen 步骤：

```
noelemahcz@nixos ~/llvm-project/build> ninja -t targets | rg tablegen
...
/home/noelemahcz/llvm-project/build/tools/clang/CMakeFiles/clang-tablegen-targets: phony
clang-tablegen-targets: phony
...
```

果然找到了，执行 `ninja clang-tablegen-targets` 生成所有 `.inc` 文件，问题解决。

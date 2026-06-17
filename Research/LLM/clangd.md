---
tags:
  - 编译原理
  - LLVM
---
源码目录位于 `clang-tool-extra/clangd` ，主函数在 `tool/ClangdToolMain.cpp` 中。其中引入 `ClangdMain.cpp` 中的 `clang::clangd::clangdMain()` ，这里是主函数的实现核心。其中主要是在处理外围参数，容易找到核心在 `ClangdLSPServer.cpp` 中。

这里有一个类 `ClangdLSPServer` ，其中的 `run` 函数调用了 `Transp.loop` ，可以猜测这里面是核心逻辑。
---
tags:
  - C
  - Cpp
  - 编译工具
---

# CMake 常用命令

CMake 是跨平台的 C/C++ 构建系统生成器，与 [[Makefile语法|Makefile]] 配合使用。`compile_commands.json` 是编译数据库文件，可供 LSP、clangd 等工具实现代码跳转和补全。

## 生成 compile_commands.json

```sh
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .
```

## 相关笔记

- [[Makefile语法]]
- [[clang 编译器语法]]
- [[Android.bp文件格式]]
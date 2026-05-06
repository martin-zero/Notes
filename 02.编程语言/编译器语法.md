---
tags:
  - C
  - Cpp
---

	本篇以clang编译器为例，gcc与其类似

# 直接编译
编译main.c文件为main可执行文件
```sh
clang main.c -o main
```

# 分阶段编译
## 预处理
预处理main.c 生成文件main.i
```sh
clang -E main.c > main.i
```

## 生成汇编
生成main.c编译后汇编main.s
```sh
clang -S main.c -o main.s
```

## 生成目标文件
编译main.c生成目标文件main.o
```sh
clang -c main.c -o main.o
```

## 链接生成可执行文件
链接main.o文件 生成可执行文件main
```sh
clang main.o -o main
```

# 调试
使用`-g` 添加调试信息以供gdb、lldb调试
```sh
clang -g main.c -o main
```
---
tags:
  - C
  - Cpp
  - 编译工具
date: 2026-05-06
---
> Makefile文件用于描述C/C++工程的编译规则，用来自动化编译大型C/C++项目。

# 语法
## 规则
```Makefile
targets : prerequisites; [command]
	command
```
- targets：规则目标，可以是 Object File（.o中间文件），也可以是可执行文件，还可以是一个标签；
- prerequisites：依赖文件，要生成 targets 需要的文件或者是目标。可以是多个，用空格隔开，也可以是没有；
- command：make 需要执行的命令（任意的 shell 命令）。可以有多条命令，每一条命令占一行；
如果 command 太长, 可以用 \ 作为换行符。

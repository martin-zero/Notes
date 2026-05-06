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
Makefile由若干条规则构成，每条规则指出一个目标文件和若干个依赖文件，以及生成目标文件的命令。
当`targets`的创建时间晚于`prerequisites`的最后修改时间不会触发对应的`command`。
```Makefile
targets: prerequisites1 prerequisites2
	command
```
- `targets：`  目标文件，可以是 Object File（.o中间文件），也可以是可执行文件，还可以是一个标签；
- `prerequisites：`依赖文件，生成 targets 需要的文件或者是目标。可以是多个，用空格隔开，也可以是没有；
- `command：`make 需要执行的命令（任意的 shell 命令）。可以有多条命令，每一条命令占一行；如果 command 太长, 可以用 \ 作为换行符。

> [!注意]
>  在Makefile的规则中，命令必须以Tab开头，不能是空格。



## 变量

## 伪目标

```Makefile
targets:
	command
```
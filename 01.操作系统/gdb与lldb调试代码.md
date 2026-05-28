---
tags:
  - 调试
---
## 启动与退出

| 操作 | GDB | LLDB |
|------|-----|------|
| 启动调试 | `gdb ./prog` | `lldb ./prog` |
| 带参数启动 | `gdb --args ./prog arg1 arg2` | `lldb -- ./prog arg1 arg2` |
| 附加到进程 | `gdb -p <pid>` | `lldb -p <pid>` |
| 加载 core dump | `gdb ./prog core` | `lldb -c core ./prog` |
| 退出 | `quit` / `q` | `quit` / `q` |

## 运行控制

| 操作 | GDB | LLDB |
|------|-----|------|
| 运行 | `run` / `r` | `run` / `r` |
| 继续 | `continue` / `c` | `continue` / `c` |
| 单步进入函数 | `step` / `s` | `step` / `s` |
| 单步跳过函数 | `next` / `n` | `next` / `n` |
| 执行到返回 | `finish` / `fin` | `finish` / `fin` |
| 单条汇编步入 | `stepi` / `si` | `stepi` / `si` |
| 单条汇编跳过 | `nexti` / `ni` | `nexti` / `ni` |
| 执行到指定行 | `until <line>` | `thread until <line>` |
| 终止程序 | `kill` | `kill` |

## 断点

| 操作 | GDB | LLDB |
|------|-----|------|
| 函数断点 | `break foo` / `b foo` | `breakpoint set -n foo` / `b foo` |
| 行断点 | `break file.c:42` / `b file.c:42` | `b file.c:42` |
| 地址断点 | `b *0x7fff5fbff000` | `b -a 0x7fff5fbff000` |
| 条件断点 | `b foo if x > 10` | `b foo -c 'x > 10'` |
| 临时断点 | `tbreak foo` / `tb foo` | `b foo -o` |
| 列出断点 | `info breakpoints` / `i b` | `breakpoint list` / `br list` |
| 禁用断点 | `disable <num>` | `breakpoint disable <id>` |
| 启用断点 | `enable <num>` | `breakpoint enable <id>` |
| 删除断点 | `delete <num>` / `d <num>` | `breakpoint delete <id>` |
| 删除全部断点 | `delete` | `breakpoint delete` |
| 忽略断点 N 次 | `ignore <num> <count>` | `b foo -i 5` |
| 断点命令列表 | `commands <num>` | `breakpoint command add <id>` |

## 监视点

| 操作 | GDB | LLDB |
|------|-----|------|
| 监视变量写入 | `watch x` | `watchpoint set variable x` / `wa s v x` |
| 监视变量读取 | `rwatch x` | `watchpoint set variable x -r` |
| 监视地址 | `watch *0x600000` | `watchpoint set expression -a 0x600000` |
| 条件监视 | `watch x if x > 100` | `wa s v x -c 'x > 100'` |
| 列出监视点 | `info watchpoints` | `watchpoint list` |

## 查看变量与内存

| 操作 | GDB | LLDB |
|------|-----|------|
| 打印变量 | `print x` / `p x` | `print x` / `p x` |
| 打印表达式 | `p x + y` | `p x + y` |
| 打印结构体 | `p *ptr` | `p *ptr` |
| 查看类型 | `ptype var` | `type lookup var` |
| 查看局部变量 | `info locals` | `frame variable` / `fr v` |
| 查看函数参数 | `info args` | `fr v` |
| 查看寄存器 | `info registers` / `i r` | `register read` / `re r` |
| 查看特定寄存器 | `p $rax` | `re r rax` |
| 修改寄存器 | `set $rax = 42` | `register write rax 42` |
| 查看内存 (hex) | `x/20x $rsp` | `memory read --size 4 --format x $rsp` |
| 查看内存 (字符串) | `x/s addr` | `memory read addr` |
| 查看内存 (指令) | `x/10i $rip` | `dis -s $rip -c 10` |
| 自动显示 | `display x` | `target stop-hook add -o "p x"` |

## 数据格式化

格式说明符，放在 `p/` 后面：

| 格式 | 说明 | 示例 |
|------|------|------|
| `x` | 十六进制 | `p/x 255` → `0xff` |
| `d` | 十进制有符号 | `p/d -1` |
| `u` | 十进制无符号 | `p/u -1` |
| `o` | 八进制 | `p/o 8` → `010` |
| `t` | 二进制 | `p/t 5` → `0b101` |
| `a` | 地址 | `p/a ptr` |
| `c` | 字符 | `p/c 65` → `'A'` |
| `f` | 浮点数 | `p/f 3.14` |
| `s` | 字符串 | `p (char*)addr` |

GDB `x/` 内存查看格式：`x/[数量][格式][大小] 地址`

大小：`b`=1字节, `h`=2字节, `w`=4字节, `g`=8字节

例：`x/10gx $rsp` — 从 rsp 开始打印 10 个 8 字节单元，十六进制

## 调用栈

| 操作 | GDB | LLDB |
|------|-----|------|
| 查看调用栈 | `backtrace` / `bt` | `thread backtrace` / `bt` |
| 查看 N 层 | `bt 10` | `bt -c 10` |
| 查看全部线程栈 | `thread apply all bt` | `thread backtrace all` |
| 切换栈帧 | `frame <n>` / `f <n>` | `frame select <n>` / `f <n>` |
| 上移栈帧 | `up` | `up` |
| 下移栈帧 | `down` | `down` |
| 查看当前帧源码 | `list` / `l` | `source list` |
| 查看当前帧信息 | `info frame` | `frame info` |

## 线程

| 操作 | GDB | LLDB |
|------|-----|------|
| 列出线程 | `info threads` | `thread list` |
| 切换线程 | `thread <n>` | `thread select <n>` |
| 对所有线程执行 | `thread apply all <cmd>` | `thread backtrace all` |

## 汇编与反汇编

| 操作 | GDB | LLDB |
|------|-----|------|
| 反汇编当前函数 | `disassemble` / `disas` | `disassemble -f` / `dis` |
| 反汇编指定函数 | `disas foo` | `dis -n foo` |
| 反汇编地址范围 | `disas 0x400000,0x400100` | `dis -s 0x400000 -e 0x400100` |
| 反汇编 N 条指令 | `x/10i $rip` | `dis -s $rip -c 10` |
| 显示当前指令 | `x/i $rip` | `dis -s $rip -c 1` |
| 切换 Intel 风格 | `set disassembly-flavor intel` | `settings set target.x86-disassembly-flavor intel` |

## 多进程调试

| 操作 | GDB | LLDB |
|------|-----|------|
| fork 后跟随子进程 | `set follow-fork-mode child` | `settings set target.process.follow-fork-mode child` |
| 不 detach 父进程 | `set detach-on-fork off` | — |
| 查看所有子进程 | `info inferiors` | `platform process list` |

## 反向调试 (GDB)

需要先 `record` 开启记录。

| 命令 | 说明 |
|------|------|
| `reverse-continue` / `rc` | 反向继续执行 |
| `reverse-step` / `rs` | 反向单步进入 |
| `reverse-next` / `rn` | 反向单步跳过 |
| `reverse-finish` | 反向执行到调用点 |

## 配置文件

### .gdbinit
```
set pagination off
set print pretty on
set print array on
set print array-indexes on
set disassembly-flavor intel
set history save on
set follow-fork-mode child
```

### .lldbinit
```
settings set stop-disassembly-count 0
settings set target.x86-disassembly-flavor intel
settings set target.process.follow-fork-mode child
```

## TUI / GUI 模式

GDB：
- `Ctrl-x a` — 切换 TUI
- `Ctrl-x 1` — 单窗口
- `Ctrl-x 2` — 双窗口（源码 + 汇编）
- `tui reg general` — 显示寄存器窗口
- `Ctrl-l` — 刷新屏幕

LLDB：输入 `gui` 进入 curses 界面。

## 常见调试场景

### 段错误 (SIGSEGV)

```gdb
gdb ./prog core
(gdb) bt              # 看崩溃栈
(gdb) frame 0         # 切到崩溃帧
(gdb) info registers  # 看寄存器（哪个指针非法）
(gdb) p ptr           # 打印可疑指针
(gdb) x/s ptr         # 尝试读指针内容
```

### 调试运行中的进程

```bash
gdb -p $(pidof myapp)
lldb -p $(pidof myapp)
```

### 条件断点（循环中 i==100 时停下）

```gdb
(gdb) b loop.c:10 if i == 100
```
```lldb
(lldb) b loop.c:10 -c 'i == 100'
```

### 断点触发后自动执行命令

```gdb
(gdb) b foo
(gdb) commands
> p x
> p y
> continue
> end
```
```lldb
(lldb) b foo
(lldb) breakpoint command add
> p x
> p y
> continue
> DONE
```

### addr2line 定位源码行

```bash
addr2line -e ./prog -f -C 0x400123
```

### Android native 调试

```bash
# 目标设备
adb shell ps -A | grep <进程名>
adb shell gdbserver64 :5039 --attach <pid>
adb forward tcp:5039 tcp:5039

# 主机
gdb-multiarch ./symbols/lib.so
(gdb) target remote :5039
```

### LLDB 远程调试

```bash
# 目标机
lldb-server platform --server --listen *:12345

# 主机
(lldb) platform select remote-linux
(lldb) platform connect connect://target:12345
```

## 命令对照规律

GDB 用 `info <thing>` 查信息，LLDB 用 `<thing> list` 结构化子命令：

| 类别 | GDB | LLDB |
|------|-----|------|
| 断点 | `info b` | `br list` |
| 线程 | `info threads` | `thread list` |
| 寄存器 | `info registers` | `register read` / `re r` |
| 局部变量 | `info locals` | `frame variable` / `fr v` |
| 内存 | `x/` + 格式串 | `memory read` |
| 反汇编 | `disassemble` | `disassemble` / `dis` |

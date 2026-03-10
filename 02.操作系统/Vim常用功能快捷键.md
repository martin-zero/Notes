---
tags:
  - 工具/neovim
---
# 常规
`^` 跳转到本行开头
`$` 跳转到本行结尾
`*` 跳转到光标下单词的下一个匹配
`#` 跳转到光标下单词的上一个匹配


# LazyVim
### 搜索
`<leader> sb` 搜索当前文件匹配行

### 跳转

`gd` 跳转到定义
`gD` 跳转到声明
`gI` 跳转到实现
`gy` 跳转到类型定义
`gr` 查找所有引用
`<leader>ch` 切换头文件与源文件

### 提示

`K` 悬停提示
`<C-k>` 函数参数提示
`gK` 签名提示

### 诊断

`[d` 跳到上一个诊断
`]d` 跳到下一个诊断
`<leader>cd` 查看当前行诊断信息
`<leader>xx` 打开Trouble面板查看所有诊断

### 重构

`<leader>cr` 符号重命名
`<leader>ca` 快速修复(小灯泡)

### 格式化

`<leader>cf` 格式化当前文件
`<leader> cF` 格式化选中区域

### 工作区

`<leader>cs` 当前文件的符号(函数/类/变量)
`<leader>cS` 全工程符号搜索
`<leader>cD` 跳转到类型定义

## debug

`<leader>db` 切换断点
`<leader>dB` 条件断点
`<leader>dc` 启动调试
`<leader>dp` 打印变量

## 窗口管理

`<leader>|` 垂直分屏
`<leader>-` 水平分屏
`<leader>wd` 关闭当前窗口
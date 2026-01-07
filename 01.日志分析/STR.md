---
tags:
  - 技术栈/Android/Framework
  - 子系统/STR
date: 2025-11-28
---
# 排查方法
 1. 可以通过qnx_dump/la_gvm或qnx_log/cycle_xxx_x/LaGvm.txt文件中搜索关键字`suspend`定位阻止休眠进程号。LaGvm文件上的时间+8为Android侧对应时间。

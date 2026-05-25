---
tags:
  - Linux日用
date: 2026-05-23
---
使用`debtap`工具可以将`.deb`转化为`.pkg.tar.zst`格式的，之后使用pacman -U进行安装。
```sh
# 更新 debtap 数据库（首次使用） 
sudo debtap -u
# 转换 .deb 包 
sudo debtap package.deb
# 安装转换后的包 
sudo pacman -U package.pkg.tar.zst
```
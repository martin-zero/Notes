---
tags:
  - Linux日用
date: 2026-05-25
---

# 1. 清理 pacman 缓存

查看缓存大小：

```
du -sh /var/cache/pacman/pkg
```

安装缓存清理工具：

```
sudo pacman -S pacman-contrib
```

保留最近 3 个版本：

```
sudo paccache -r
```

只保留当前安装版本：

```
sudo paccache -rk1
```

删除所有未安装缓存：

```
sudo paccache -ruk0
```

彻底清空缓存（不推荐）：

```
sudo pacman -Scc
```

---

# 2. 删除 orphan（孤儿包）

查看孤儿包：

```
pacman -Qdt
```

删除孤儿包：

```
sudo pacman -Rns $(pacman -Qdtq)
```

---

# 3. 清理 AUR 缓存（yay）

如果使用 yay：

清理无用依赖：

```
yay -Yc
```

清理构建缓存：

```
yay -Sc
```

彻底清理：

```
yay -Scc
```

手动删除缓存：

```
rm -rf ~/.cache/yay/*
```

查看缓存大小：

```
du -sh ~/.cache/yay
```

---

# 4. 清理 systemd 日志

查看日志占用：

```
journalctl --disk-usage
```

仅保留 7 天日志：

```
sudo journalctl --vacuum-time=7d
```

限制日志大小：

```
sudo journalctl --vacuum-size=500M
```

---

# 5. 清理用户缓存

查看缓存：

```
du -sh ~/.cache/*
```

清理缓存：

```
rm -rf ~/.cache/*
```

说明：

- 不会删除配置
- 浏览器/缩略图会重新生成缓存

---

# 6. 清理缩略图缓存

```
rm -rf ~/.cache/thumbnails/*
```

---

# 7. 查找大文件

扫描系统：

```
ncdu /
```

扫描 home：

```
ncdu ~
```

---

# 8. 清理临时文件

安全清理：

```
systemd-tmpfiles --clean
```

强制清理：

```
sudo rm -rf /tmp/*
```

---

# 9. 清理旧内核

查看内核：

```
pacman -Q | grep linux
```

查看当前使用内核：

```
uname -r
```

删除不用的内核：

```
sudo pacman -Rns linux-lts
```
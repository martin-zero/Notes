---
tags:
  - Android系统
---

# Android cmdline 参数说明

Android 的 **cmdline（kernel command line）** 是在内核启动时由 bootloader 传递给 Linux 内核的一组参数，主要用于控制系统启动行为、硬件初始化以及 Android 特有功能（如 SELinux、分区挂载等）。

这些参数通常可以在 `/proc/cmdline` 中查看。

> [!TODO] 待补充
> - [ ] 常用 cmdline 参数列表
> - [ ] `androidboot.` 前缀参数说明

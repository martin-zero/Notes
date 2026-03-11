---
tags:
  - 技术栈/Android/Framework
---
![](assets/Android启动流程/file-20260310153327161.png)

## Kernel
引导阶段、Linux Kernel启动阶段见[Linux启动流程](../01.嵌入式/Linux启动流程.md)。
#TODO 差异化 

## Init
Init进程是Android用户空间的第一个进程，进程号为1，其main.cpp的代码路径为[`system/core/init/main.cpp`](https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp;l=1?q=init%2Fmain&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn)，main函数中init分为三个阶段：
·FirstStage阶段负责**初始化系统基础环境**与**挂载核心文件系统**。
SetupSelinux阶段负责**初始化并启用Selinux安全机制**。
SecondStage为Init流程中最重要的一个阶段，这个阶段主要负责**解析init.rc文件，并根据init.rc文件启动Android系统Native层的服务**，其中`servicemanager`与`zygote`就是在这里起来的。
完成以上步骤后init不会退出，而是进入了一个事件循环。主要监听**property 变化**，**service 进程状态**等事件。

## Zygote


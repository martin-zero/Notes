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
- FirstStage阶段负责**初始化系统基础环境**与**挂载核心文件系统**。
- SetupSelinux阶段负责**初始化并启用Selinux安全机制**。
- SecondStage为Init流程中最重要的一个阶段，这个阶段主要负责**解析init.rc文件，并根据init.rc文件启动Android系统native层的服务**，其中`servicemanager`与`zygote`就是在这里起来的。
- 完成以上步骤后init不会退出，而是进入了一个事件循环。主要监听**property 变化**，**service 进程状态**等事件。

## Zygote
Zygote进程是在init.rc文件中声明，由init进程启动的，在Android系统的运行过程中起着非常重要的作用，**Java层的所有进程都是由Zygote fork出来的**。init进程显示通过解析init.rc文件执行[`frameworks/base/cmds/app_process/app_main.cpp`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/cmds/app_process/app_main.cpp;l=1;bpv=0;bpt=1?q=App_main.cpp&sq=&ss=android%2Fplatform%2Fsuperproject%2Fmain&hl=zh-cn)中的main，app_process的app_main负责**创建和初始化JVM虚拟机、准备Java环境**，之后调用其Java层下的main函数，它位于[`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;l=91?q=ZygoteIn&sq=&hl=zh-cn)目录下。

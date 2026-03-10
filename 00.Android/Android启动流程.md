---
tags:
  - 技术栈/Android/Framework
---
![](assets/Android启动流程/file-20260310153327161.png)

## Kernel
引导阶段、Linux Kernel启动阶段见[Linux启动流程](../01.嵌入式/Linux启动流程.md)。

## Init
Init进程是Android用户空间的第一个进程，进程号为1，其main函数的代码路径为[`system/core/init/main.cpp`](https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp;l=1?q=init%2Fmain&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn)。


---
tags:
  - 技术栈/Android/Framework
---
![](assets/Android启动流程/file-20260310153327161.png)

## Kernel
引导阶段、Linux Kernel启动阶段见[Linux启动流程](../01.嵌入式/Linux启动流程.md)

## Init
Init进程是Android用户空间的第一个进程，进程号为1，代码路径为`system/core/init/main.cpp`。
Init内部执行细分了三个阶段: FirstStageMain、SetupSelinux、SecondStageMain。
### FirstStageMain


## Zygote

## SystemServer



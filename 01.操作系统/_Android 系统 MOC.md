---
tags:
  - MOC
  - Android系统
---
![Android数据库](../10.数据库/Android数据库.base)

# Android 系统 — 笔记索引

## 启动流程

- [[Android启动流程]] — 完整启动链路概览（Kernel → Init → Zygote → SystemServer → Launcher）
- [[Init 进程]] — 用户空间第一个进程，解析 init.rc，拉起所有 native 服务
- [[Zygote 进程]] — 所有 Java 进程的父进程，预加载 Framework 资源
- [[SystemServer 启动]] — 引导/核心/其他服务三阶段启动，AMS/PMS/WMS 的诞生
- [[ServiceManager自举原理]] — Binder 驱动钦点的句柄 0，比 Zygote 还早

## Binder 与 IPC

- [[ServiceManager自举原理]] — 内核级"后门"解决自举问题
- [[Android智能指针]] — sp/wp，侵入式引用计数，Binder 引用管理的基础

## Activity 与窗口

- [[Android Activity冷启动全链路]] — 6 次跨进程通信，4 个进程，8 个常见误区

## 构建系统

- [[Android.mk]] — 传统 Makefile 风格的 AOSP 模块定义
- [[Android.bp文件格式]] — Soong 构建系统的类 JSON 蓝图文件
- [[Android分区与镜像]] — boot/system/vendor/userdata 等 9 种分区镜像
- [[Android系统级应用打包流程]] — 从源码到 /system/priv-app 的完整步骤

## 调试

- [[Android系统代码调试方法]] — jdb（Java Framework）+ lldb（Native）调试流程
- [[gdb、lldb速查表]] — GDB/LLDB 命令对照速查

## 系统参数与目录

- [[Android cmdline参数说明]] — 内核启动参数，androidboot.* 前缀
- [[Android目录结构]] — AOSP 源码目录树

## 项目实战

- [[ROC-RK3328-PC编译、烧录、调试]] — Firefly 开发板完整流程
- [[吉利cc值修改]] — 车机 CC 配置调试方法
- [[吉利QNX模拟车身信号]] — VehicleTool 绕过台架验证
- [[吉利日志分析思路与步骤]] — 蓝牙/USB/音频/休眠/ANR 各模块日志关键字

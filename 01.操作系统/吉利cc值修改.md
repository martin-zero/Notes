---
tags:
  - 吉利
---

# 吉利 CC 值修改

通过 adb 命令行临时修改 CC（Car Configuration）配置的方法，仅供开发调试使用。

## 临时修改 CC（SOC 侧）

通过 adb 连接车机，使用 telnet 进入 QNX 侧执行：

```sh
su
busybox telnet 192.168.118.2
root
test_psis_car_cfg CFG_CAR_CONFIGURATION xx x
```

> [!WARNING] 仅临时测试
> 该命令只修改 SOC 侧的 CC 配置，未修改 SCC 侧的 CC 配置。
> 只适用于开发快速验证功能，不可用于正式台架测试。

## 退出测试模式

正式台架测试时，应当从台架输入 CC 数据到 SCC 侧，并退出 SOC 的测试模式：

```sh
psis_client -w -1 hut_cfg CFG_DEBUG_MODE 0
bosch_reset
```

> [!IMPORTANT] 注意事项
> 台架输入 CC 和手动命令行输入 CC 不能混用。
> UM=Abandoned 再发送 UM=Driving 后，SCC 会覆盖当前 SOC 的 CC 配置，这是正常的流程。

## 相关笔记

- [[吉利QNX模拟车身信号]]
- [[吉利日志分析思路与步骤]]

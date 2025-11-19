---
tags:
  - 休眠
---
该问题是想让Android调查是否正确执行str休眠流程，我们通过`PowerManagerService`与`DisplayPowerController`关键字确认异常发生时间点附近的休眠执行情况。


> [!NOTE] 指南
> PowerManagerService:负责管理休眠、唤醒、屏幕、亮度和节能策略。
> DisplayPowerController:直接控制屏幕的开关、亮度、休眠等状态。

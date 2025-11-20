---
tags:
  - 休眠
---
## 问题描述
```
Precondition  
台架压测  
Operation  
1.STR压测  
Observation  

STR压测时偶发enter str fail  
串口打印：[qcore io_devctl:1644] Client "display_client.0" sent in band ack (rc=16,state=suspend)  
Expectation  
无异常现象  
Clearing  
  
Notes  
BUILD_ID=rb-geely-idc20_oneos_kx11-a3-sop_bosch_dev_2025.43.8  
BUILD_TIMESTAMP=2025-10-26 11:19:00 UTC  
MANIFESTS_REVISION=a3d95ada64a45dcc88602e5778bff61c13f74d41  
MANIFEST_FILE=manifest.xml  
BUILD_MODE=remake  
TARGET_PRODUCT=8155  
TARGET_BUILD_VARIANT=userdebug  
SECURE_BOOT=rbxc  
BUILD_TAG=  
MAIN_VERSION=rb-geely-idc20_oneos_kx11-a3-sop_bosch_dev_2025.43.8  
SOC_VERSION=rb-geely-idc20_oneos_kx11-a3-sop_bosch_dev_2025.43.8  
PROJECT_ID=ECARX_DHU  
DATE_TIME_LONG=  


测试人员：xu zhixin

Test Setup & Environment
台架 
Software Configuration & Navigation Data Carrier:  
RAS0525441254411  
rb-geely-idc20_oneos_kx11-a3-sop_bosch_dev_2025.43.8  
Hardware Sample Revision:
GLY_VAVE3_128G_外置  
Test Vehicle/Bench Setup:

Used Test Media & Phones:
```

## 分析过程
该问题是想让Android调查是否正确执行str休眠流程，我们通过`PowerManagerService`与`DisplayPowerController`关键字确认异常发生时间点附近的休眠执行情况。
```c
2000-01-31 17:36:11.749   749   749 E DisplayPowerController: failed to set up display white-balance: java.lang.IllegalStateException: cannot find light sensor
2000-01-31 17:50:48.048   749  2744 I PowerManagerService: Going to sleep due to application (uid 1000)...
2000-01-31 17:50:48.051   749  1191 I PowerManagerService: Dozing...
2000-01-31 17:50:48.273   749  1191 I PowerManagerService: handleSandman now: 920752 ( Process.SYSTEM_UID: 1000)....
2000-01-31 17:50:48.273   749  1191 I PowerManagerService: reallyGoToSleepNoUpdateLocked: eventTime=920752, uid=1000
2000-01-31 17:50:48.274   749  1191 I PowerManagerService: Sleeping (uid 1000)...
2000-01-31 17:50:48.558   749  1779 I PowerManagerService: Waking up from Asleep (uid=1000, reason=WAKE_REASON_APPLICATION, details=android.server.wm:SCREEN_ON_FLAG)...
2000-01-31 17:50:48.575   749  1191 D DisplayPowerController: setScreenState state= 2, reportOnly= false
2000-01-31 17:50:48.679   749  1191 D DisplayPowerController: setScreenState state= 2, reportOnly= false
2000-01-31 17:50:48.680   749  1191 D DisplayPowerController: setScreenState state= 2, reportOnly= false
```

以下日志代表系统开始进入STR:
```
2000-01-31 17:50:48.274   749  1191 I PowerManagerService: Sleeping (uid 1000)...
```

以下日志代表系统正在被唤醒:
```
2000-01-31 17:50:48.558   749  1779 I PowerManagerService: Waking up from Asleep (uid=1000, reason=WAKE_REASON_APPLICATION, details=android.server.wm:SCREEN_ON_FLAG)...
```
所以可以得出系统在刚进入STR284ms后就马上被唤醒了。

我们再看被唤醒周围的代码:
```c
2000-01-31 17:50:48.556   749  1779 D StatusBarManagerService: onWindowChange() called with: args = [Bundle[{systemUiVisibility=1280, type=1, title=com.geely.permission.service/com.geely.permission.service.ui.ShowPermissionDialogActivity, displayId=0, packageName=com.geely.permission.service}]]
2000-01-31 17:50:48.556   749  1779 D StatusBarManagerService: onWindowChange() called with: args = [Bundle[{systemUiVisibility=1280, type=1, title=com.geely.permission.service/com.geely.permission.service.ui.ShowPermissionDialogActivity, displayId=0, packageName=com.geely.permission.service}]]
2000-01-31 17:50:48.556   749  1779 D StatusBarManagerService: onWindowChange() called with: args = [Bundle[{systemUiVisibility=0, type=3, title=Splash Screen com.geely.provision, displayId=2, packageName=com.geely.provision}]]
2000-01-31 17:50:48.558  4724  4832 I LinkService: UiInteractionManager: onWindowVisibilityChanged visibility: 0 Package: com.geely.permission.service display id: 0 windowTag: com.geely.permission.service/com.geely.permission.service.ui.ShowPermissionDialogActivity
2000-01-31 17:50:48.557   749  1779 D StatusBarManagerService: onWindowChange() called with: args = [Bundle[{systemUiVisibility=0, type=3, title=Splash Screen com.geely.provision, displayId=2, packageName=com.geely.provision}]]
2000-01-31 17:50:48.558   749  1779 I PowerManagerService: Waking up from Asleep (uid=1000, reason=WAKE_REASON_APPLICATION, details=android.server.wm:SCREEN_ON_FLAG)...
```
可以得出是由于`com.geely.permission.service`发出弹窗导致了窗口可见性发生变化，触发亮屏机制。

> [!NOTE] 解释
> PowerManagerService:负责管理休眠、唤醒、屏幕、亮度和节能策略。
> DisplayPowerController:直接控制屏幕的开关、亮度、休眠等状态。


## 结论
休眠失败是由于com.geely.permission.service发出权限申请弹窗，导致窗口可见性发生变化，触发亮屏机制，属于Andorid正常的系统机制。
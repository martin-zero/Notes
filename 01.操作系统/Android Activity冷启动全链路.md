---
tags:
  - Android系统
---

## 核心一句话

> 冷启动一个 Activity 至少 **6 次跨进程通信**（5 次 Binder + 1 次 socket），经过 4 个进程：Launcher、system_server、Zygote、新 App 进程。核心转折点是 `attachApplication`——新进程通过它向 AMS 报到，AMS 才知道该往哪发 `LaunchActivityItem`。

---

## 8 个常见误区

### 误区一：「Activity 启动是 AMS 管的」

**真相**：Android 10 之后归 **ATMS**（ActivityTaskManagerService）管。

- AMS 现在只管：进程生命周期、oom_adj、ContentProvider、Broadcast
- ATMS 管：Activity 启动、Task 栈、Display
- 面试还说"走 Binder 到 AMS"会被扣分

---

### 误区二：「Launcher 直接调 ATMS.startActivity」

**真相**：中间隔了一个 `Instrumentation`。

```
Activity.startActivity()
  → Activity.startActivityForResult()
    → Instrumentation.execStartActivity()
      → ActivityTaskManager.getService().startActivity()  // Binder #1
```

`Instrumentation` 是所有 Activity 生命周期的统一拦截点，可被测试框架替换。

---

### 误区三：「启动只有一两次 Binder 调用」

**真相**：冷启动至少 6 次跨进程通信。

|#|方向|调用|作用|
|---|---|---|---|
|1|Launcher → system_server|`ATMS.startActivity()`|发起启动请求|
|2|system_server → Launcher|`ClientTransaction(PauseActivityItem)`|让 Launcher pause|
|3|Launcher → system_server|`ATMS.activityPaused()`|报告 pause 完成|
|4|system_server → Zygote|socket 通信（非 Binder）|请求 fork 新进程|
|5|新进程 → system_server|`AMS.attachApplication()`|新进程报到|
|6|system_server → 新进程|`ClientTransaction(LaunchActivityItem)`|通知启动 Activity|

**为什么要先 pause 旧 Activity？** Android 强制约束：同一时刻只有一个 Activity 处于 resumed 状态，否则输入事件不知该发给谁。

---

### 误区四：「fork 完进程就直接跑 Activity 了」

**真相**：fork 出来的进程不知道要启动什么，要先跑 `ActivityThread.main()`，然后主动向 AMS 报到。

java

```java
// ActivityThread.java
public static void main(String[] args) {
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);  // → AMS.attachApplication()  Binder #5
    Looper.loop();  // 永不返回
}
```

`mAppThread`（ApplicationThread）是新进程暴露给 system_server 的 Binder 接口，AMS 拿到它才能向新进程发后续命令。

---

### 误区五：「AMS 调 scheduleLaunchActivity 启动 Activity」

**真相**：Android 9 之后改用 **ClientTransaction** 机制，`scheduleLaunchActivity` 已废弃。

java

```java
// 一个事务同时打包多个指令
clientTransaction.addCallback(LaunchActivityItem.obtain(...));
clientTransaction.setLifecycleStateRequest(ResumeActivityItem.obtain(...));
```

原因：旧版每个生命周期变更是独立 Binder 调用，事务化后可以合并，减少跨进程开销。

---

### 误区六：「Application.onCreate 之前没别的事」

**真相**：`handleBindApplication` 的完整顺序：

1. 创建 `LoadedApk`
2. 创建 `ContextImpl`
3. 创建 `Instrumentation`
4. 反射创建 `Application` 对象
5. `Application.attachBaseContext()`
6. **`installContentProviders()`** ← **ContentProvider.onCreate 在这里执行**
7. `Application.onCreate()`

⚠️ **经典坑**：`ContentProvider.onCreate()` 早于 `Application.onCreate()`，不能在 Provider 里访问 Application 中才初始化的字段，否则 NPE。

---

### 误区七：「performLaunchActivity 就是调 onCreate」

**真相**：`performLaunchActivity` 做了 6 件事：

1. 取出 `ComponentName`
2. 创建 `ContextImpl`
3. 反射创建 `Activity` 实例
4. 创建或获取 `Application`
5. **`Activity.attach()`** ← Window 在这里创建，`getWindow()` 从此可用
6. `Instrumentation.callActivityOnCreate()` ← 终于到 `onCreate`

---

### 误区八：「冷/热启动区别只是有没有 fork」

**真相**：整个调用链路都不同。

|阶段|冷启动|热启动|
|---|---|---|
|Binder #1|✅|✅|
|Pause 旧 Activity|✅|✅|
|fork 进程|✅|❌|
|attachApplication|✅|❌|
|handleBindApplication|✅|❌|
|ContentProvider 初始化|✅|❌|
|LaunchActivityItem|✅|✅（走 ResumeActivityItem）|

Pixel 7 参考数据：冷启动 300–500ms，热启动 50–100ms。

---

## 完整调用链（白板手撕版）

```
[Launcher]
  startActivity → Instrumentation.execStartActivity
    → ATMS.startActivity                          Binder #1 → [system_server]

[system_server]
  ActivityStarter.execute → startActivityInner
    → pause Launcher (ClientTransaction)          Binder #2 → [Launcher]

[Launcher]
  handlePauseActivity → Activity.onPause
    → ATMS.activityPaused                         Binder #3 → [system_server]

[system_server]
  Process.start                                   socket   → [Zygote] → fork

[新进程]
  ActivityThread.main → Looper.prepareMainLooper
    → thread.attach → AMS.attachApplication       Binder #5 → [system_server]

[system_server]
  attachApplicationLocked
    → bindApplication (ClientTransaction)         Binder #5.1 → [新进程]
    → realStartActivityLocked
      → LaunchActivityItem (ClientTransaction)    Binder #6 → [新进程]

[新进程]
  handleBindApplication
    → 创建 Application → ContentProvider.onCreate → Application.onCreate
  handleLaunchActivity → performLaunchActivity
    → Activity.attach → Activity.onCreate
  handleResumeActivity → Activity.onResume        ← 用户看到界面
```

---

## 高频追问

**Q：为什么要先 pause 旧 Activity？** SurfaceFlinger 的 z-order 管理要求同一时刻只有一个 resumed Activity，否则输入焦点无法确定。

**Q：attachApplication 的作用？** 新进程把自己的 `ApplicationThread`（Binder 回调接口）注册给 AMS，AMS 从此能向它发命令。

**Q：fork 后、attach 前进程 crash 了怎样？** AMS 有 `PROC_START_TIMEOUT`（10 秒）超时，超时后清理 `ProcessRecord` 和 `ActivityRecord`，通知 WMS 移除窗口，用户看到"无响应"或回到 Launcher。

**Q：ContentProvider.onCreate 和 Application.onCreate 谁先？** ContentProvider 先。在 `installContentProviders` 阶段执行，早于 `Application.onCreate`。

**Q：Android 10 为什么拆出 ATMS？** 职责分离，AMS 原来代码超 2 万行。ATMS 放入 `com.android.server.wm` 包，暗示未来 Activity 管理与窗口管理可能进一步合并。
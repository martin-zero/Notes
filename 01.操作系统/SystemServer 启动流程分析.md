---
tags:
  - Android系统/systemserver
  - 流程分析
---

# SystemServer 启动

> SystemServer 是 Android Java 世界的核心，**所有系统服务（AMS、PMS、WMS 等）都在这里启动**。它由 [[Zygote 流程分析|Zygote]] fork 出来后通过反射运行 `SystemServer.main()`。没有 SystemServer，应用无法启动、窗口无法显示、广播无法分发。

Zygote fork SystemServer 后反射运行 [frameworks/base/services/java/com/android/server/SystemServer.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/java/com/android/server/SystemServer.java;l=329;drc=61197364367c9e404c7da6900658f1b16c42d0da;bpv=0;bpt=1?q=SystemServer&sq=&ss=android%2Fplatform%2Fsuperproject%2Fmain&hl=zh-cn) 的 `main` 方法，创建 SystemServer 对象并执行其 `run` 方法。

```java
private void run() {
        TimingsTraceAndSlog t = new TimingsTraceAndSlog();
        try {
            t.traceBegin("InitBeforeStartServices");

            // 记录启动信息
            SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
            SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
            SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));

            EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START,
                    mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);

            // 时区与语言初始化
            SystemTimeZone.initializeTimeZoneSettingsIfRequired();


            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();

                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }

            Binder.setWarnOnBlocking(true);
            PackageItemInfo.forceSafeLabels();

            // SQLite 初始化
            SQLiteGlobal.sDefaultSyncMode = SQLiteGlobal.SYNC_MODE_FULL;

            SQLiteCompatibilityWalFlags.init(null);

            Slog.i(TAG, "Entered the Android system server!");

...

        // 启动系统服务（按三个阶段）
        try {
            t.traceBegin("StartServices");
            startBootstrapServices(t);    // 关键服务：AMS、PMS、PowerManager 等
            startCoreServices(t);         // 核心服务：BatteryService、UsageStats 等
            startOtherServices(t);        // 其他服务：WMS、InputManager、AudioService 等
            startApexServices(t);
            // 初始化看门狗
            updateWatchdogTimeout(t);
            CriticalEventLog.getInstance().logSystemServerStarted();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            t.traceEnd(); // StartServices
        }

...

        // 进入主循环
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

    private static boolean isValidTimeZoneId(String timezoneProperty) {
        return timezoneProperty != null
                && !timezoneProperty.isEmpty()
                && ZoneInfoDb.getInstance().hasTimeZone(timezoneProperty);
    }

    private boolean isFirstBootOrUpgrade() {
        return mPackageManagerService.isFirstBoot() || mPackageManagerService.isDeviceUpgrading();
    }

    private void reportWtf(String msg, Throwable e) {
        Slog.w(TAG, "***********************************************");
        Slog.wtf(TAG, "BOOT FAILURE " + msg, e);
    }
```

## 服务启动的三个阶段

| 阶段 | 方法 | 包含的关键服务 |
|------|------|---------------|
| 引导服务 | `startBootstrapServices` | AMS、PMS、PowerManager、DisplayManager |
| 核心服务 | `startCoreServices` | BatteryService、UsageStatsService、WebViewUpdate |
| 其他服务 | `startOtherServices` | WMS、InputManager、AudioService、NotificationManager |

---
tags:
  - Android系统/zygote
  - 流程分析
---

# Zygote 进程

> Zygote 是 Android 中**所有 Java 进程的父进程**。它在 init.rc 中声明，由 [[Android Init 流程分析|Init]] 拉起，负责预加载 Framework 资源并 fork 出 [[SystemServer 启动流程分析|SystemServer]]。之后进入无限循环，通过 socket 等待 AMS 的 fork 指令来创建应用进程。

## app_process

Init 通过解析 init.rc 执行 [frameworks/base/cmds/app_process/app_main.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/cmds/app_process/app_main.cpp;l=1;bpv=0;bpt=1?q=App_main.cpp&sq=&ss=android%2Fplatform%2Fsuperproject%2Fmain&hl=zh-cn) 的 `main`。app_process 负责**创建 JVM 虚拟机、注册 JNI、准备 Java 环境**。

```cpp
int main(int argc, char* const argv[])
{

    ...

    // 初始化虚拟机
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));

...

    bool known_command = false;

    // 读取并解析命令参数
    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
 
 ...

    // 解析运行模式
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = (arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className = arg;
            break;
        } else {
            --i;
            break;
        }
    }

...

    // 启动 Java 层 ZygoteInit 的 main 方法
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.empty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}

```

## ZygoteInit

从这里开始正式进入 Java 层。ZygoteInit 位于 [frameworks/base/core/java/com/android/internal/os/ZygoteInit.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;l=91?q=ZygoteIn&sq=&hl=zh-cn)，负责**预加载系统资源**（Framework classes、常用资源文件、字体及共享库等），使 fork 出的进程无需重复加载即可使用。

环境就绪后，Zygote **fork 出 SystemServer 进程**，然后**进入无限循环通过 ZygoteServer 的 socket 等待 fork 指令**。

```java
 public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;

        // 禁止 Zygote 在 fork 前创建线程，避免锁状态错乱
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

...

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                // 预加载资源，fork 后的进程直接可用（核心优化）
                preload(bootTimingsTraceLog);

...

            // 创建 socket 服务器，用于接收 fork 指令
            zygoteServer = new ZygoteServer(isPrimaryZygote);
            // fork SystemServer 进程
            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} 在父进程（zygote），{@code r != null} 在子进程（system_server）
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // 无限循环，等待 AMS 的 fork 指令
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with fatal exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // 只在子进程中执行（AMS fork 出的应用进程）
        if (caller != null) {
            caller.run();
        }
    }
```

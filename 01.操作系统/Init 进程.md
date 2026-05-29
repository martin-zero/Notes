---
tags:
  - Android系统
  - Android系统/init
---

# Init 进程

> Init 是 Android 用户空间的第一个进程（PID 1），负责挂载文件系统、启用 SELinux、解析 init.rc 并拉起所有 native 服务（包括 [[ServiceManager自举原理|ServiceManager]] 和 [[Zygote 进程|Zygote]]）。理解 Init 是理解 Android 启动全链路的起点。

代码路径：[system/core/init/main.cpp](https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp;l=1?q=init%2Fmain&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn)

`main` 函数根据参数分为三个阶段：

```cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#elif __has_feature(hwaddress_sanitizer)
    __hwasan_set_error_report_callback(AsanReportCallback);
#endif
    setpriority(PRIO_PROCESS, 0, -20);
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }
    return FirstStageMain(argc, argv);
}
```

## FirstStage

负责**初始化系统基础环境**与**挂载核心文件系统**。

```cpp
int FirstStageMain(int argc, char** argv) {
    
    ...
    
    // 创建并挂载最基础的内核接口
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);

    umask(0);

    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
   
... 

#undef CHECKCALL

    // 将标准输入输出重定向为null
    SetStdioToDevNull(argv);
    // 初始化内核日志
    InitKernelLogging(argv);
    
    // 输出错误日志
    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

    // 第一阶段初始化开始
    LOG(INFO) << "init first stage started!";

...

    // 决定启动参数 (normal boot, recovery boot, charger mode)
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;

    // 加载 kernel modules
    boot_clock::time_point module_start_time = boot_clock::now();
    int module_count = 0;
    BootMode boot_mode = GetBootMode(cmdline, bootconfig);
    if (!LoadKernelModules(boot_mode, want_console,
                           want_parallel, module_count)) {
        if (want_console != FirstStageConsoleParam::DISABLED) {
            LOG(ERROR) << "Failed to load kernel modules, starting console";
        } else {
            LOG(FATAL) << "Failed to load kernel modules";
        }
    }
    if (module_count > 0) {
        auto module_elapse_time = std::chrono::duration_cast<std::chrono::milliseconds>(
                boot_clock::now() - module_start_time);
        setenv(kEnvInitModuleDurationMs, std::to_string(module_elapse_time.count()).c_str(), 1);
        LOG(INFO) << "Loaded " << module_count << " kernel modules took "
                  << module_elapse_time.count() << " ms";
    }

    MaybeResumeFromHibernation(bootconfig);

    // 在/dev目录下创建必要设备节点
    std::unique_ptr<FirstStageMount> fsm;

    bool created_devices = false;
    if (want_console == FirstStageConsoleParam::CONSOLE_ON_FAILURE) {
        if (!IsRecoveryMode()) {
            fsm = CreateFirstStageMount(cmdline);
            if (fsm) {
                created_devices = fsm->DoCreateDevices();
                if (!created_devices) {
                    LOG(ERROR) << "Failed to create device nodes early";
                }
            }
        }
        StartConsole(cmdline);
    }

...

        // 挂载系统分区
        if (!fsm->DoFirstStageMount()) {
            LOG(FATAL) << "Failed to mount required partitions early ...";
        }
    }

...

    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
#ifdef HWASAN_OPTIONS
    setenv("HWASAN_OPTIONS", STRINGIFY(HWASAN_OPTIONS), true);
#endif
    // 进入 selinux_setup 阶段
    execv(path, const_cast<char**>(args));

    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```

## SetupSelinux

负责**初始化并启用 SELinux 安全机制**。

```cpp
int SetupSelinux(char** argv) {

    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    SelinuxSetupKernelLogging();

    if (IsMicrodroid()) {
        LoadSelinuxPolicyMicrodroid();
    } else {
        LoadSelinuxPolicyAndroid();
    }

    SelinuxSetEnforcement();

    if (IsMicrodroid() && android::virtualization::IsOpenDiceChangesFlagEnabled()) {
        const int flags = SELINUX_ANDROID_RESTORECON_RECURSE;
        if (selinux_android_restorecon("/microdroid_resources", flags) == -1) {
            PLOG(FATAL) << "restorecon of /microdroid_resources failed";
        }
    }

    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);

    SetupOverlays();

    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```

## SecondStage

Init 流程中最重要的阶段，负责**解析 init.rc 文件并启动 native 层服务**。[[ServiceManager自举原理|ServiceManager]] 与 [[Zygote 进程|Zygote]] 就在这里被拉起。

完成后 init 进入事件循环，监听 **property 变化**、**service 进程状态**等事件。

```cpp
int SecondStageMain(int argc, char** argv) {

...

    // 解析 init.rc 文件，启动系统服务
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);

...
    
    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    // 死循环，进行事件监听
    setpriority(PRIO_PROCESS, 0, 0);
    while (true) {
        const boot_clock::time_point far_future = boot_clock::time_point::max();
        boot_clock::time_point next_action_time = far_future;

        auto shutdown_command = shutdown_state.CheckShutdown();
        if (shutdown_command) {
            LOG(INFO) << "Got shutdown_command '" << *shutdown_command
                      << "' Calling HandlePowerctlMessage()";
            HandlePowerctlMessage(*shutdown_command);
        }

        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
            if (am.HasMoreCommands()) {
                next_action_time = boot_clock::now();
            }
        }
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();

            if (next_process_action_time) {
                next_action_time = std::min(next_action_time, *next_process_action_time);
            }
        }

        std::optional<std::chrono::milliseconds> epoll_timeout;
        if (next_action_time != far_future) {
            epoll_timeout = std::chrono::ceil<std::chrono::milliseconds>(
                    std::max(next_action_time - boot_clock::now(), 0ns));
        }
        auto epoll_result = epoll.Wait(epoll_timeout);
        if (!epoll_result.ok()) {
            LOG(ERROR) << epoll_result.error();
        }
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }

    return 0;
}
```

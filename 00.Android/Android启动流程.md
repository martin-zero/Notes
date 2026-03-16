---
tags:
  - 技术栈/Android/Framework
---
![](assets/Android启动流程/file-20260310153327161.png)

# Kernel
引导阶段、Linux Kernel启动阶段见[Linux启动流程](../01.嵌入式/Linux启动流程.md)。
#TODO 差异化 

# Init
Init进程是Android用户空间的第一个进程，进程号为1，其main.cpp的代码路径为[`system/core/init/main.cpp`](https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp;l=1?q=init%2Fmain&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn)，main函数中init分为三个阶段：
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

## 
FirstStage阶段主要负责**初始化系统基础环境**与**挂载核心文件系统**。
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
        // 打印终止日志并终止进程(LOG(FATAL)会触发abort)
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

	// 第一阶段初始化开始
    LOG(INFO) << "init first stage started!";

...

	// 决定启动参数 (normal boot, recovery boot, charger mode)
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;

	// 加载 kernel modules
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
	// 进入selinux_setup阶段
    execv(path, const_cast<char**>(args));

    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```

SetupSelinux阶段负责**初始化并启用Selinux安全机制**。
```cpp
int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    SelinuxSetupKernelLogging();

    // TODO(b/287206497): refactor into different headers to only include what we need.
    if (IsMicrodroid()) {
        LoadSelinuxPolicyMicrodroid();
    } else {
        LoadSelinuxPolicyAndroid();
    }

    SelinuxSetEnforcement();

    if (IsMicrodroid() && android::virtualization::IsOpenDiceChangesFlagEnabled()) {
        // We run restorecon of /microdroid_resources while we are still in kernel context to avoid
        // granting init `tmpfs:file relabelfrom` capability.
        const int flags = SELINUX_ANDROID_RESTORECON_RECURSE;
        if (selinux_android_restorecon("/microdroid_resources", flags) == -1) {
            PLOG(FATAL) << "restorecon of /microdroid_resources failed";
        }
    }

    // We're in the kernel domain and want to transition to the init domain.  File systems that
    // store SELabels in their xattrs, such as ext4 do not need an explicit restorecon here,
    // but other file systems do.  In particular, this is needed for ramdisks such as the
    // recovery image for A/B devices.
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);

    // SetupOverlays does not return if overlays exist, instead it execs overlay_remounter
    // which then execs second stage init
    SetupOverlays();

    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never return from this function.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```
SecondStage为Init流程中最重要的一个阶段，这个阶段主要负责**解析init.rc文件，并根据init.rc文件启动Android系统native层的服务**，其中`servicemanager`与`zygote`就是在这里起来的。
完成以上步骤后init不会退出，而是进入了一个事件循环。主要监听**property 变化**，**service 进程状态**等事件。

# Zygote
Zygote进程是在init.rc文件中声明，由init进程启动的，在Android系统的运行过程中起着非常重要的作用，**Java层的所有进程都是由Zygote fork出来的**。
init进程显示通过解析init.rc文件执行[`frameworks/base/cmds/app_process/app_main.cpp`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/cmds/app_process/app_main.cpp;l=1;bpv=0;bpt=1?q=App_main.cpp&sq=&ss=android%2Fplatform%2Fsuperproject%2Fmain&hl=zh-cn)中的main，
app_process的app_main负责**创建JVM虚拟机、注册JNI、准备Java环境**。

app_main将环境准备好后会执行它Java层ZygoteInit的main函数，它位于[`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;l=91?q=ZygoteIn&sq=&hl=zh-cn)目录下，
---
tags:
  - 技术栈/Android/Framework
---
![](assets/Android启动流程/file-20260310153327161.png)

## Kernel
引导阶段、Linux Kernel启动阶段见[Linux启动流程](../01.嵌入式/Linux启动流程.md)。
#TODO 差异化 

## Init
Init进程是Android用户空间的第一个进程，进程号为1，其main.cpp的代码路径为[`system/core/init/main.cpp`](https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp;l=1?q=init%2Fmain&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn)，main函数中init分为三个阶段：
```cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#elif __has_feature(hwaddress_sanitizer)
    __hwasan_set_error_report_callback(AsanReportCallback);
#endif
    // Boost prio which will be restored later
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

FirstStage阶段负责**初始化系统基础环境**与**挂载核心文件系统**。
```cpp
int FirstStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    std::vector<std::pair<std::string, int>> errors;
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);

    // 清空权限掩码，让mkdir直接设置文件权限
    umask(0);

	// 初始化环境、创建并挂载核心文件系统
    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
   ... 


    SetStdioToDevNull(argv);
    // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
    // talk to the outside world...
    InitKernelLogging(argv);

    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

    LOG(INFO) << "init first stage started!";

    auto old_root_dir = std::unique_ptr<DIR, decltype(&closedir)>{opendir("/"), closedir};
    if (!old_root_dir) {
        PLOG(ERROR) << "Could not opendir(\"/\"), not freeing ramdisk";
    }

    struct stat old_root_info {};
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;

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

    if (access(kBootImageRamdiskProp, F_OK) == 0) {
        std::string dest = GetRamdiskPropForSecondStage();
        std::string dir = android::base::Dirname(dest);
        std::error_code ec;
        if (!fs::create_directories(dir, ec) && !!ec) {
            LOG(FATAL) << "Can't mkdir " << dir << ": " << ec.message();
        }
        if (!fs::copy_file(kBootImageRamdiskProp, dest, ec)) {
            LOG(FATAL) << "Can't copy " << kBootImageRamdiskProp << " to " << dest << ": "
                       << ec.message();
        }
        LOG(INFO) << "Copied ramdisk prop to " << dest;
    }

    // If "/force_debuggable" is present, the second-stage init will use a userdebug
    // sepolicy and load adb_debug.prop to allow adb root, if the device is unlocked.
    if (access("/force_debuggable", F_OK) == 0) {
        constexpr const char adb_debug_prop_src[] = "/adb_debug.prop";
        constexpr const char userdebug_plat_sepolicy_cil_src[] = "/userdebug_plat_sepolicy.cil";
        std::error_code ec;  // to invoke the overloaded copy_file() that won't throw.
        if (access(adb_debug_prop_src, F_OK) == 0 &&
            !fs::copy_file(adb_debug_prop_src, kDebugRamdiskProp, ec)) {
            LOG(WARNING) << "Can't copy " << adb_debug_prop_src << " to " << kDebugRamdiskProp
                         << ": " << ec.message();
        }
        if (access(userdebug_plat_sepolicy_cil_src, F_OK) == 0 &&
            !fs::copy_file(userdebug_plat_sepolicy_cil_src, kDebugRamdiskSEPolicy, ec)) {
            LOG(WARNING) << "Can't copy " << userdebug_plat_sepolicy_cil_src << " to "
                         << kDebugRamdiskSEPolicy << ": " << ec.message();
        }
        // setenv for second-stage init to read above kDebugRamdisk* files.
        setenv("INIT_FORCE_DEBUGGABLE", "true", 1);
    }

    if (ForceNormalBoot(cmdline, bootconfig)) {
        mkdir("/first_stage_ramdisk", 0755);
        PrepareSwitchRoot();
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            PLOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }

    if (IsRecoveryMode()) {
        LOG(INFO) << "First stage mount skipped (recovery mode)";
    } else {
        if (!fsm) {
            fsm = CreateFirstStageMount(cmdline);
        }
        if (!fsm) {
            LOG(FATAL) << "FirstStageMount not available";
        }

        if (!created_devices && !fsm->DoCreateDevices()) {
            LOG(FATAL) << "Failed to create devices required for first stage mount";
        }

        if (!fsm->DoFirstStageMount()) {
            LOG(FATAL) << "Failed to mount required partitions early ...";
        }
    }

    struct stat new_root_info {};
    if (stat("/", &new_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    if (old_root_dir && old_root_info.st_dev != new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }

    SetInitAvbVersionInRecovery();

    setenv(kEnvFirstStageStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(),
           1);

    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
#ifdef HWASAN_OPTIONS
    // (second stage) init is the first process to use HWASan. It gets run too
    // early for the HWASAN_OPTIONS in init.environ.rc.gen to apply, so we
    // have to do it here manually.
    //
    // This is especially important for disable_coredump, which defaults to 1.
    // If the user sets it to `0` in HWASAN_OPTIONS, and we didn't also apply
    // it to (second stage) init, the setting would not take effect. This is
    // because `0` is in fact a noop, while `1` applies changes. These changes
    // are inherited beyond exec.
    setenv("HWASAN_OPTIONS", STRINGIFY(HWASAN_OPTIONS), true);
#endif
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```
SetupSelinux阶段负责**初始化并启用Selinux安全机制**。
SecondStage为Init流程中最重要的一个阶段，这个阶段主要负责**解析init.rc文件，并根据init.rc文件启动Android系统native层的服务**，其中`servicemanager`与`zygote`就是在这里起来的。
完成以上步骤后init不会退出，而是进入了一个事件循环。主要监听**property 变化**，**service 进程状态**等事件。

## Zygote
Zygote进程是在init.rc文件中声明，由init进程启动的，在Android系统的运行过程中起着非常重要的作用，**Java层的所有进程都是由Zygote fork出来的**。
init进程显示通过解析init.rc文件执行[`frameworks/base/cmds/app_process/app_main.cpp`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/cmds/app_process/app_main.cpp;l=1;bpv=0;bpt=1?q=App_main.cpp&sq=&ss=android%2Fplatform%2Fsuperproject%2Fmain&hl=zh-cn)中的main，
app_process的app_main负责**创建JVM虚拟机、注册JNI、准备Java环境**。

app_main将环境准备好后会执行它Java层ZygoteInit的main函数，它位于[`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;l=91?q=ZygoteIn&sq=&hl=zh-cn)目录下，
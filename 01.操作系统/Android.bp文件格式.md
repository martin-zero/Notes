---
tags:
  - Android系统
---

# Android.bp 文件格式

> 官方文档：[Android.bp Reference](https://source.android.com/docs/setup/reference/androidbp?hl=zh-cn)

> 在 AOSP 中，Android.bp 是使用 Soong 构建系统 的蓝图文件（Blueprint），用于替代老旧的 Android.mk。它采用的是 JSON 类似的 结构化声明方式，用于描述模块的构建方式，例如 app、库、jar、C/C++ 模块等。

## 整体结构

```bp
<module_type> {
    name: "模块名称",
    srcs: ["源文件1", "源文件2", ...],
    ...
}

```

`<module_type>`: 构建模块类型，多个模块可以在一个Android.bp文件中声明，常见类型如下表所示。

|模块类型|说明|
|---|---|
|`android_app`|构建 Android APK 应用|
|`java_library`|构建 Java 静态库|
|`cc_binary`|构建 C/C++ 可执行程序|
|`cc_library`|构建 C/C++ 动态或静态库|
|`prebuilt_jar`|使用预编译 jar|
|`filegroup`|定义一组文件集合，可供引用|
|`genrule`|通用构建规则，可调用脚本或命令行|

## 构成字段介绍

### App模块(android_app)

```bp
android_app {
    name: "MySystemApp",             // 模块名，最终 apk 输出名为 MySystemApp.apk
    srcs: ["src/**/*.java", "src/**/*.kt"], // Java, Kotlin 源文件路径
    resource_dirs: ["res"],          // 资源文件目录
    manifest: "AndroidManifest.xml", // 指定 Manifest 文件
    sdk_version: "system_current",   // 使用系统 SDK
    privileged: true,                // 安装到 /system/priv-app（具备系统权限）
    certificate: "platform",         // 使用 platform key 签名
    platform_apis: true,             // 允许使用系统 API
    dex_preopt: {
        enabled: true,               // 允许 dex 优化
    },
    aaptflags: ["", ...],            // aapt 编译时附加参数

	static_libs: [                   // 链接的java静态库
        "androidx.room_room-runtime",
        "androidx.lifecycle_livedata-core",
        "androidx.lifecycle_viewmodel",
        "androidx.sqlite_sqlite",
        "kotlin-stdlib",
    ],

    libs: ["", ...],                 // 运行时依赖库
    optimize.enabled: false,         // 是否启用Proguard
    multilib: "64",                  // C/C++ 模块用，指定编译架构（例如：both、32、64）

    databinding: {
        enabled: true,               // 启用DataBinding
    },

	aidl: {                          // AIDL接口
        include_dirs: ["src"],
    },

}

```

### 预编译文件模块（prebuilt)

```bp
prebuilt_etc {
    name: "my_config_xml",
    src: "my_config.xml",
    sub_dir: "my_config",
    installable: true,
}

```

### 文件组模块（filegroup）

用于收集一组文件，供其他模块引用（如 aidl、proto）：

```bp
filegroup {
    name: "my_aidl_files",
    srcs: ["src/**/*.aidl"],
    path: "src",
}

```
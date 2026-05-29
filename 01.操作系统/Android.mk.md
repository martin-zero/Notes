---
tags:
  - Android系统/编译系统
date: 2026-05-28
---

# Android.mk

> Android.mk 是 AOSP 传统 Makefile 风格的模块定义方式，其继任者 [[Android.bp文件格式|Android.bp]] 采用类 JSON 声明式语法。

# 常用变量
- **LOCAL_PATH := \$(call my-dir):** 当前模块路径，通常直接使用`$(call my-dir)`。
- **include $(CLEAR_VARS) :** 清除之前定义的所有`LOCAL_`变量，防止影响当前模块，固定语法。
- **LOCAL_MODULE:** 模块名称。
- **LOCAL_MODULE_TAGS:** 模块标签。
- **LOCAL_SRC_FILES:** 源文件。
- **LOCAL_C_INCLUDES:** C头文件路径。
- **LOCAL_CFLAGS:** C编译器标志(例如`-WALL -O2`)。
- **LOCAL_CPPFLAGS:** C++编译器标志（例如`-std=c++11`）。
- **LOCAL_STATIC_LIBRARIES:** 静态库依赖。
- **LOCAL_SHARED_LIBRARIES:** 动态库依赖。
- **LOCAL_LDLIBS:** 链接库(例如`-llog`表示链接`liblog.so`)。
- **LOCAL_PRELINK_MODULE:** 是否预链接(bool类型)。
- **LOCAL_COPY_HEADERS:** 导出头文件。
- **LOCAL_COPY_HEADERS_TO:** 头文件输出目录。

# 示例
```Android.mk
# 指定当前AndroiAndroid.mk路径
LOCAL_PATH := $(call my-dir)

# 清空之前定义的所有LOCAL_局部变量，防止影响当前模块
include $(CLEAR_VARS) 

# 编译模块名称 
LOCAL_MODULE := framework 

# 指定源文件 
LOCAL_SRC_FILES := \
    src/main.cpp \
    src/util.cpp

# 指定C头文件路径
LOCAL_C_INCLUDES := $(LOCAL_PATH)/include

# 动态库依赖
LOCAL_SHARED_LIBRARIES := \
    libutils \
    libcutils

# 编译器标志
LOCAL_CFLAGS := -Wall -Werror

# 编译为动态库
include $(BUILD_SHARED_LIBRARY)
```


> [!TODO]
> 在实际开发中遇到补充
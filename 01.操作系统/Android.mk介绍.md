# 基本结构

```mk
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := my_module_name
LOCAL_SRC_FILES := my_source_file.c

include $(BUILD_SHARED_LIBRARY)
```
- **LOCAL_PATH:** 当前模块路径，通常直接使用`$(call my-dir)`。
- **include $(CLEAR_VARS) :** 清除之前定义的所有`LOCAL_`变量，防止影响当前模块，固定语法。
- **LOCAL_MODULE:** 定义模块名称。
- **LOCAL_SRC_FILES:** 定义模块的源文件。
- **include $(BUILD_SHARED_LIBRARY):** 编译为动态库

# 
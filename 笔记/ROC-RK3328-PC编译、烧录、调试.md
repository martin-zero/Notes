---
tags:
  - 技术栈/AndroidFramework
---


>[!链接]
>官网Wiki：https://wiki.t-firefly.com/zh_CN/ROC-RK3328-PC/android_adb.html
>


# 编译
## 32位整编
```shell
./FFTools/make.sh -d roc-rk3328-pc -j8 -l roc_rk3328_pc_32-userdebug
./FFTools/mkupdate/mkupdate.sh -l roc_rk3328_pc_32-userdebug
```

## 模块化编译
### 编译内核
```shell
 ./FFTools/make.sh -b -j8 
```

### 编译U-Boot
```shell
./FFTools/make.sh -u -j8
```

### 编译Android
```shell
./FFTools/make.sh -a -j8
```

### 编译全部分区
```shell
./FFTools/make.sh -j8
```


## 不使用脚本编译
### 配置环境变量
```shell
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```
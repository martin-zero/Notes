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


## 编译内核
```shell
 ./FFTools/make.sh -b -j8 
```

## 编译U-Boot
```shell
./FFTools/make.sh -u -j8
```

## 编译Android
```shell
./FFTools/make.sh -a -j8
```

## 编译全部分区
```shell
./FFTools/make.sh -j8
```


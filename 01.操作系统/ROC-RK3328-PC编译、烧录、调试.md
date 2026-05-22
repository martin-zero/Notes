---
tags:
  - Android系统
---

# ROC-RK3328-PC 编译、烧录、调试

Firefly ROC-RK3328-PC 开发板的 AOSP 编译、固件烧录与调试环境搭建记录。包含脚本编译和手动分步编译两种方式。

> [!INFO] 参考链接
> 官网Wiki：https://wiki.t-firefly.com/zh_CN/ROC-RK3328-PC/android_adb.html


# 编译
## 32位整编
```sh
./FFTools/make.sh -d roc-rk3328-pc -j8 -l roc_rk3328_pc_32-userdebug
./FFTools/mkupdate/mkupdate.sh -l roc_rk3328_pc_32-userdebug
```

## 模块化编译
### 编译内核
```sh
 ./FFTools/make.sh -b -j8 
```

### 编译U-Boot
```sh
./FFTools/make.sh -u -j8
```

### 编译Android
```sh
./FFTools/make.sh -a -j8
```

### 编译全部分区
```sh
./FFTools/make.sh -j8
```


## 不使用脚本编译
### 配置环境变量
```sh
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

### 编译内核
```sh
make ARCH=arm64 firefly_defconfig android-10.config
make ARCH=arm64 -j8 BOOT_IMG=../rockdev/Image-roc_rk3328_pc_32/boot.img roc-rk3328-pc.img
```

### 编译 U-Boot
```sh
./make.sh rk3328
```

### 编译 Android
```sh
source build/envsetup.sh
lunch roc_rk3328_pc_32-userdebug
make installclean
make -j8
./mkimage.sh
```


# 烧录
## Android10使用upgrade_tool烧录各个模块
```sh
sudo upgrade_tool di -b boot.img
sudo upgrade_tool di -dtbo dtbo.img  
sudo upgrade_tool di -misc misc.img
sudo upgrade_tool di -parameter parameter.txt
sudo upgrade_tool di -r recovery.img
sudo upgrade_tool di -super super.img
sudo upgrade_tool di -trust trust.img
sudo upgrade_tool di -uboot uboot.img
sudo upgrade_tool di -vbmeta vbmeta.img
```


# 调试
## scrcpy 
GitHub: https://github.com/Genymobile/scrcpy
用于在电脑上显示和控制您的安卓设备


### 串口
```sh
picocom -b 1500000 /dev/ttyUSB0
```

### Dockerfile
```dockerfile
FROM ubuntu:18.04

ENV DEBIAN_FRONTEND=noninteractive

RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
    sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y \
    openjdk-8-jdk \
    git-core gnupg flex bison gperf build-essential \
    zip curl libncurses5-dev zlib1g-dev gcc-multilib g++-multilib \
    libc6-dev-i386 lib32ncurses5-dev lib32z1-dev lib32readline-dev\
    libssl-dev xsltproc unzip python python-dev python3 python3-pip bc swig liblz4-tool rsync && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /aosp && \
    useradd -m -s /bin/bash build

USER build

WORKDIR /aosp

CMD ["bash"]

```
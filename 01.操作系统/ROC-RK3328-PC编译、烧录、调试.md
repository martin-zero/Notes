---
tags:
  - 技术栈/Android/Framework
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

### 编译内核
```shell
make ARCH=arm64 firefly_defconfig android-10.config
make ARCH=arm64 -j8 BOOT_IMG=../rockdev/Image-roc_rk3328_pc_32/boot.img roc-rk3328-pc.img
```

### 编译 U-Boot
```shell
./make.sh rk3328
```

### 编译 Android
```shell
source build/envsetup.sh
lunch roc_rk3328_pc_32-userdebug
make installclean
make -j8
./mkimage.sh
```


# 烧录
## Android10使用upgrade_tool烧录各个模块
```SHELL
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
```
picocom -b 1500000 /dev/ttyUSB0
```

### Dockerfile
```dockerfile
FROM ubuntu:14.04

# 设置环境变量
ENV DEBIAN_FRONTEND=noninteractive

# 安装JDK 8
RUN add-apt-repository ppa:openjdk-r/ppa && \
    apt-get update && \
    apt-get install -y openjdk-8-jdk
    
# 更新系统并安装必要的包
RUN  apt-get install -y \
    git-core gnupg flex bison gperf libsdl1.2-dev \
    libesd0-dev libwxgtk2.8-dev squashfs-tools build-essential zip curl \
    libncurses5-dev zlib1g-dev pngcrush schedtool libxml2 libxml2-utils \
    xsltproc lzop libc6-dev schedtool g++-multilib lib32z1-dev lib32ncurses5-dev \
    lib32readline-gplv2-dev gcc-multilib libswitch-perl \
    libssl1.0.0 libssl-dev p7zip-full \
    software-properties-common swig python-dev bc


# 清理缓存，减少镜像大小
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# 创建工作目录
WORKDIR /aosp

# 设置默认命令
CMD ["bash"]
```
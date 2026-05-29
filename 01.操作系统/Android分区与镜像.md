---
tags:
  - Android系统/编译系统
---
# 常见镜像介绍

1. **boot.img-> boot 分区**：启动镜像，包含内核和启动引导程序。
2. **system.img -> system 分区**：系统镜像，包含 Android 系统的核心组件和系统应用。
3. **vendor.img-> vendor 分区**：供应商镜像，包含设备厂商提供的硬件相关库和驱动。
4. **recovery.img -> recovery 分区**：恢复镜像，提供系统恢复功能，允许重置和更新系统。
5. **userdata.img-> userdata 分区**：用户数据镜像，包含用户的应用和数据文件。
6. **cache.img-> cache 分区**：缓存镜像，用于系统的缓存文件。
7. **dtbo.img -> dtbo 分区**：设备树分区镜像，包含设备硬件配置信息。
8. **vbmeta.img-> vbmeta 分区**：用于验证镜像的完整性和安全性（Android Verified Boot）。
9. **product.img-> product分区**：存放设备厂商或产品特定的应用、库以及其他数据文件。
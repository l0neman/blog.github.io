---
title: Dropbear Android 安装步骤
toc: true
date: 2021-04-21 23:04:39
categories:
- Android 实用工具
tags:
- Android
- SSH
---

1. 克隆 `https://github.com/ubiquiti/dropbear-android` 仓库
2. 修改 `build-dropbear-android.sh` 文件中的编译器路径：

```shell
HOST=arm-linux-androideabi
COMPILER=${TOOLCHAIN}/bin/armv7a-linux-androideabi28-clang
STRIP=${TOOLCHAIN}/bin/arm-linux-androideabi-strip
SYSROOT=${TOOLCHAIN}/sysroot
```
<!-- more -->

3. 设置 `TOOLCHAIN` 变量：

```shell
export TOOLCHAIN=/xxx/ndk/22.0.7026061/toolchains/llvm/prebuilt/linux-x86_64
```

4. 编译执行 `build-dropbear-android.sh` 脚本

生成了 `target/dropbear` 和 `target/dropbearkey` 可执行文件


5. 把两个文件放入 Android 设备 `/system/xbin/` 中

6. 生成密钥：

```shell
dropbearkey -t dss -f /system/etc/dropbear/dropbear_dss_host_key
dropbearkey -t rsa -f /system/etc/dropbear/dropbear_rsa_host_key
```

7. 启动服务并设置用户名和密码：

```shell
dropbear -A -N root -C 123456
```

8. 客户端连接：

```shell
ssh root@10.2.17.16 -p 10022
```

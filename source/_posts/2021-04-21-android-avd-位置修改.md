---
title: Android AVD 位置修改
toc: true
date: 2021-04-21 18:20:58
categories:
- Android 实用工具
tags:
- AVD
---
## 前言

默认的 Android Virtual Device Manager (AVD)，就是官方 Android 模拟器的镜像文件存储位置在 `C:\Users\<user_name>\.android\avd` 中，Linux 和 Mac 在 `<user_home>/.android/avd` 中，有时需要改变它的位置，例如给 C 盘腾出空间。

Windows 系统中操作步骤如下：
<!-- more -->


## 操作步骤

1. 关闭 Android Studio；
2. 控制面板 -> 系统 -> 高级系统设置 -> 环境变量；
3. 添加一个用户变量：
 - 变量名：`ANDROID_SDK_HOME`
 - 变量值：`<更改后的位置>`，例如 `D:\Tool\Avd`，其中更改后的位置不能是 Android SDK 的根目录，可以是它的子目录
4. 打开 Android Studio，确认 `.android` 文件夹在更改后的的位置中被创建了；
5. 从以前的位置 `C:\Users\<user_name>\.android\avd` 将 `avd` 文件夹移动到新的位置 `D:\Tool\Avd` 中

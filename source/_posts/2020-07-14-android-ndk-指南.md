---
title: Android NDK 指南
toc: true
date: 2020-07-14 23:20:17
tags:
- Android
- NDK
- JNI
categories:
- Android 应用开发
---
# 前言

编写此文档的用意：

1. 作为搭建基础 NDK 工程的教程；
2. 作为入门 NDK 工程的参考手册。



# NDK 工程构建

可采用三种方式进行 NDK 工程的构建：

1. 基于 Make 的 ndk-build，这是传统的 ndk-build 构建方式，使用 Makefile 方式进行构建，简洁高效；
2. CMake 是新型的构建方式，CMake 具有跨平台的特性，通过 CMake 生成 Makefile 后再进行构建，CMake 的配置文件可读性更高；
3. 其他编译系统，通过引入其他编译系统可对编译过程进行定制，例如引入 obfuscator-llvm 对源码进行混淆和压缩，增强源代码安全性。

下面是每种构建方式的基础示例，使用 Android Studio 4.0 和 NDK 21 进行如下构建。
<!-- more -->


## Android.mk

基于 Android.mk 的产物为 libfoo.so 的 NDK 基本工程搭建。

在 Android 工程的 src/main 下建立 jni 目录（Android.mk 工程的默认文件目录为 jni，也可指定其他目录进行构建，使用命令 `ndk-build -C 目录`），工程结构如下：

包含两个 .mk 文件用来描述 NDK 工程，和两个基本的 C++ 语言源文件，结构如下：

```
src/main
 |
 +-- java
 +-- jni
      |
      +-- Android.mk
      +-- Application.mk
      +-- libfoo.h
      +-- libfoo.cpp
```

在 Android Studio 的当前 Module 配置中指明 Android.mk 文件路径:

```groovy
// app-build.gradle

android {
  ...
  externalNativeBuild {
    ndkBuild {
      path 'src/main/jni/Android.mk'
    }
  }
}
```

编写 Android.mk 文件用于向 NDK 构建系统描述工程的 C/C++ 源文件以及共享库的属性。

```makefile
# Android.mk

LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

# 指定共享库名字，产出物为 libfoo.so
LOCAL_MODULE := foo
# 指定源代码文件，多个源代码文件使用空格分隔，换行在行尾使用 \
LOCAL_SRC_FILES := libfoo.cpp

include $(BUILD_SHARED_LIBRARY)
```

添加 Application.mk 用于描述 NDK 工程概要设置。

```makefile
# Application.mk

# 指定生成特定 ABI 的代码
APP_ABI := armeabi-v7a arm64-v8a
APP_OPTIM := debug
```

在 java 目录创建 Java 类，用于声明 JNI 方法，提供给其他类调用。

```java
// class io.l0neman.mkexample.NativeHandler
public class NativeHandler {

  static {
    // 加载 libfoo.so 库
    System.loadLibrary("foo");
  }

  public static native String getHello();
}
```

源代码：

```cpp
// libfoo.h

extern "C" {

#ifndef NDKTPROJECT_LIBFOO_H
#define NDKTPROJECT_LIBFOO_H

#include <jni.h>

// 注册指定 Java 层的 JNI 方法
JNIEXPORT jstring JNICALL
Java_io_l0neman_mkexample_NativeHandler_getHello(JNIEnv *env, jclass clazz);

};

#endif //NDKTPROJECT_LIBFOO_H
```



```cpp
// libfoo.cpp
#include "libfoo.h"

jstring Java_io_l0neman_mkexample_NativeHandler_getHello(JNIEnv *env, jclass clazz) {
  return env->NewStringUTF("Hello-jni");
}
```

这样的话就完成了一个基本的 NDK 工程搭建，编译后调用代码即可得到 java 字符串 `"Hello-jni"`。

```java
// MainActivity.java

String hello = NativeHandler.getHello();
```



- 提示

Android.mk 和 Application.mk 中可使用的系统变量请参考下文。

Android.mk 只是 Makefile 的片段，对于 Makefile 本身的熟悉有助于深入理解和编写 Android.mk，可参考 [Makfile 指南](/2020/07/14/makefile-指南/)


## CMake

使用 CMake 和 Android.mk 在 Android Studio 中的构建步骤类似，如下：

基于 CMake 的产出物为 libfoo.so 的 NDK 基本工程搭建。

在 Android 工程的 src/main 下建立 cpp 目录，工程结构如下：

包含一个 CMakeLists.txt 文件来描述 NDK 工程，和两个基本的 C++ 语言文件。

```
src/main
 |
 +-- java
     jni
      |
      +-- CMakeLists.txt
      +-- libfoo.h
      +-- libfoo.cpp
```

在 Android Studio 的当前 Module 配置中指明 CMakeLists.txt 文件路径:

```groovy
// app/build.gradle

android {
  ...
  externalNativeBuild {
    cmake {
      path 'src/main/cpp/CMakeLists.txt'
    }
  }
}
```

编写 CMakeLists.txt 文件用于向 NDK 构建系统描述工程的 C/C++ 源文件以及共享库的属性。

```cmake
# CMakeLists.txt

cmake_minimum_required(VERSION 3.4.3)

add_library(
        # 共享库名字，生产物为 libfoo.so
        foo
        # 编译为共享库 .so
        SHARED
        # 源代码文件，多个文件使用空格分隔或换行
        main.cpp
)
```

此时将 Android.mk 工程中的 Java 源文件 NativeHandler.java 复制过来，将 libfoo.cpp 和 libfoo.h 内容填入中即可直接编译测试。



## 独立工具链

有时编译 NDK 工程有一些特殊需求，例如对代码进行混淆，加入第三方编译器 obfuscator-llvm 对 NDK 工程进行编译。这时就需要搭建第三方工具链的编译环境，将它加入 NDK 的一般构建过程中。

下面是一个引入 obfuscator-llvm 编译器编译代码的示例。

### obfuscator-llvm 构建

环境：android-ndk-r14b，目前已知此版本可支持 obfuscator-llvm 的编译配置

ndk r14b 下载地址：[https://developer.android.google.cn/ndk/downloads/older_releases](https://developer.android.google.cn/ndk/downloads/older_releases)



- 首先下载编译器，指定最新版本的 obfuscator-llvm 分支，将仓库克隆至本地

```shell
git clone -b llvm-4.0 https://github.com/obfuscator-llvm/obfuscator.git
```

- 编译出编译器的可执行文件

过程如下，以下命令 Windows DOS 和 Linux Shell 中可通用：

1. 进入编译器仓库目录中 `cd obfuscator`；
2. 创建临时文件目录 `mkdir build`；
3. 进入临时文件目录 `cd build`；
4. 使用 CMake 生成 Makefile 或者 Vs 解决方案：

如果没有按照 CMake，可去 CMake 官网下载安装。

```shell
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_INCLUDE_TESTS=OFF ../
```

CMake 将会自动检测电脑上的编译器环境，如果是 Linux，生成 Makefile，如果 Windows 上安装了 Visual Studio，将生成解决方案文件。

5. 编译编译器源代码：

Linux 上执行：

```
make -j4
```

Windows 平台建议使用 Visual Studio 进行编译，直接打开 build 中的 LLVM.sln，然后生成解决方案（Build Solution）。

编译过程需要持续 30 分钟或更长时间，取决于电脑配置 CPU 性能。

编译过程中有可能出现错误，需要自己解决出现的不同情况，编译完成后将生成所需的 bin 和 lib 目录（Release 中）。



- 配置 NDK 环境

设原始 NDK 工具链根目录为 android-ndk-r14b。

进入 android-ndk-r14b/toolchains 目录中，复制已存在的 llvm 目录到 ollvm-4.0，Linux 使用 `cp llvm ollvm-4.0`，Windows 复制文件出现 `llvm-副本` 后重命名为 `ollvm-4.0`。

Windows 平台将上面编译出来的 bin 和 lib 放入 ollvm-4.0/prebuilt/windows-x86_64 中，Linux 平台放入 ollvm-4.0/prebuilt/linux-x86_64 中，macOS 为 ollvm-4.0/prebuilt/darwin-x86_64。

进入 android-ndk-r14b/build/core/toolchains 中，在当前目录复制出如下目录：

```
arm-linux-androideabi-clang -> arm-linux-androideabi-clang-ollvm4.0
aarch64-linux-android-clang -> aarch64-linux-android-clang-ollvm4.0
x86-clang                   -> x86-clang-ollvm4.0
x86_64-clang                -> x86_64-clang-ollvm4.0
```

修改复制后的两个目录中的 setup.mk 文件：

```
android-ndk-r14b/build/core/toolchains/arm-linux-androideabi-clang-ollvm4.0/setup.mk
android-ndk-r14b/build/core/toolchains/aarch64-linux-android-clang-ollvm4.0/setup.mk
android-ndk-r14b/build/core/toolchains/arm-linux-androideabi-clang-ollvm4.0/setup.mk
android-ndk-r14b/build/core/toolchains/x86_64-clang-ollvm4.0/setup.mk
```

将每个 `setup.mk` 中的如下内容：

```
LLVM_TOOLCHAIN_PREBUILT_ROOT := $(call get-toolchain-root,llvm)
```

替换为：

```
OLLVM_NAME := ollvm-4.0
LLVM_TOOLCHAIN_PREBUILT_ROOT := $(call get-toolchain-root,$(OLLVM_NAME))
```

此时使用 ndk-build 将可以识别编译器。复制 4 个目录的原因是为了支持编译出每种 ABI，（armeabi、armeabi-v7a、arm64-v8a，x86、x86_64）。



- 编译代码测试

进入 NDK 工程中，修改 Application.mk 和 Android.mk 如下：

```makefile
# Application.mk

APP_ABI := armeabi-v7a arm64-v8a
# 主要是此句指定编译器
NDK_TOOLCHAIN_VERSION := clang-ollvm4.0
```



```makefile
# Android.mk

LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := foo
LOCAL_SRC_FILES := libfoo.cpp

# 添加 obfuscator-llvm 支持的各种参数，伪控制流、控制流展开、指令替换
LOCAL_CFLAGS += -mllvm -bcf -mllvm -bcf_loop=3 \
                -mllvm -fla -mllvm -split \
                -mllvm -sub -mllvm -sub_loop=3

include $(BUILD_SHARED_LIBRARY)
```

在包含源代码的 jni 目录下执行配置好的 NDK r14b 中的 ndk-build 编译即可。



- 验证结果

编译后，在 libs 中将出现 ABI 目录，使用 IDA Pro 打开 libfoo.so，左侧 Functions windos 中找一个简单函数（例如 JNI_OnLoad）打开，发现程序逻辑流程已被混淆的面目全非。

左下角的 Graph overview 可以直观的看到整个函数的逻辑流程，非常复杂，无法直接了解到原始逻辑。



## 构建技巧

### 独立构建

通常 NDK 构建过程需要依赖于 Android Studio 进行清理，构建等工作。

有时需要脱离 Android Studio，例如在无界面的服务器上独立构建，那么可以直接使用 `ndk-build` 命令行进行构建。

首先确认 NDK 的环境变量（将 NDK 工具链的根路径加入系统 PATH 变量）。然后直接在 jni 目录下打开终端（Windows 为 cmd），输入 `ndk-build clean`，将自动清理产生的 obj 文件和 libs 文件。

然后执行 `ndk-build` 即可构建出所需要的 so 文件，例如 libs/arm64-v8a/libfoo.so。

如果不想在 jni 目录中构建，可使用 `-C` 选项指定路径构建 `ndk-build -C jni_new`。

其他参数可参考官方文档：[https://developer.android.google.cn/ndk/guides/ndk-build](https://developer.android.google.cn/ndk/guides/ndk-build)

- 提示

对于普通 Android Studio 中的工程，也可以使用这种方法构建。

首先把 gradle 中 Android.mk 路径配置去除。在默认的依赖配置里面可以看到，libs 目录已被加入依赖，就是说如果 libs 目录中有 so 文件，那么会被自动加入 apk 中。

```groovy
// app/build.gradle
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    ...
}
```

那么经过 `ndk-build` 构建后，可以直接运行 apk 工程，新的 libfoo.so 将被加入 apk 的 libs 目录中。

此时 Android Studio 构建和清理均不会影响 libs 中的 .so 文件，Java 代码和 NDK 开发代码可分别独立构建。



### 快速部署

对于一个主要由 native 代码构成的应用来说，修改 native 代码的动作较为频繁，如果每次都 clean 然后重新 build，再依赖于 Android studio 的运行安装会可能会比较麻烦。有时也需要依赖于其他 IDE 来构建 NDK 工程（例如使用 Visual Studio），那么可以采用如下方法：

首次构建 NDK 工程后安装运行到手机上，然后后面每次构建出 so，使用 adb 命令直接将 so 文件 push 到应用的沙盒目录下，重新启动应用进程即可使用新版的 so 文件。

```shell
adb push libfoo.so /data/data/io.l0neman.mkexample/lib/
```

注意 so 文件的架构应与当前应用采用的 ABI 对应。

不过这样做的前提是设备拥有 root 权限，也可直接使用官方的 Android 模拟器，选择下载带有 GoogleApis 的模拟器 ROM，输入如下命令即可获取 root 权限：

```
adb root
adb remount
```

之后 adb 将以 root 用户的身份运行。



# Android.mk 变量参考

## 变量命名规范

NDK 构建系统保留了如下变量名称，在定义自己的变量时尽量避免这些规则：

1. 以 `LOCAL_` 开头的名称，例如 `LOCAL_MODULE`；
2. 以 `PRIVATE_`、`NDK_` 或 `APP` 开头的名称，构建系统内部使用了这些变量名；
3. 小写名称，例如 `my-dir`，构建系统内部使用了这些变量名。

最好以 `MY_` 附加在自己的变量开头。



## NDK 定义的 include 变量

- CLEAR_VARS

此变量指向一个用于清理变量的脚本，当包含它时，会清理几乎所有的 `LOCAL_XXX` 变量，不包含 `LOCAL_PATH` 变量，一般在描述新模块之前包含。

```makefile
include $(CLEAR_VARS)
```

- BUILD_EXECUTABLE

指明构建的产出物是一个可执行文件（无文件后缀名），需要在源代码中包含一个 main 函数。通常构建可执行文件用来测试或用于其他调试工具。

```cpp
// foo.cpp
int main(int argv, char **args) {
  printf("Hello World!\n");
  return 0;
}
```



```makefile
include $(BUILD_EXECUTABLE)
```

- BUILD_SHARED_LIBRARY

指明构建的产出物是一个共享库（文件后缀为 .so），它会随着应用代码打包至 apk 中。

```
include $(BUILD_SHARED_LIBRARY)
```

- BUILD_STATIC_LIBRARY

指明构建的产出物是一个静态库（文件后缀为 .a），它不会被打包至 apk 中，只是为了被其他 native 模块引用。

- PREBUILT_SHARED_LIBRARY

用于描述预编译共享库的构建，此时 `LOCAL_SRC_FILES` 变量指向预编译库的路径。

```
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libbar.so
include $(PREBUILT_SHARED_LIBRARY)
```

- PREBUILT_STATIC_LIBRARY

用于描述预编译静态库的构建，此时 `LOCAL_SRC_FILES` 变量指向预编译库的路径。

```
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libbar.a
include $(PREBUILT_STATIC_LIBRARY)
```



## 目标信息变量

构建系统会根据 `APP_ABI` 变量（在 Application.mk 中定义）指定的每个 ABI 分别解析一次 Android.mk，如下变量将在构建系统每次解析时被重新定义值。

- TARGET_ARCH

对应 CPU 系列，为 `arm`、`arm64`、`x86`、`x86_64`。

- TARGET_PLATFORM

指向 Android API 级别号，例如 Android 5.1 对应 22。可以这样使用：

```makefile
ifeq ($(TARGET_PLATFORM),android-22)
    # ... do something ...
endif
```

- TARGET_ARCH_ABI

对应每种 CPU 对应架构的 ABI。

| CPU and architecture | Setting     |
| -------------------- | ----------- |
| ARMv7                | armeabi-v7a |
| ARMv8 AArch64        | arm64-v8a   |
| i6686                | x86         |
| x86-64               | x86_64      |

检查 ABI：

```makefile
ifeq ($(TARGET_ARCH_ABI),arm64-v8a)
  # ... do something ...
endif
```

- TARGET_ABI

目标 Android API 级别与 ABI 的串联值。检查在 Android API 级别 22 上运行的 64 位 ARM 设备：

```makefile
ifeq ($(TARGET_ABI),android-22-arm64-v8a)
  # ... do something ...
endif
```



## 模块描述变量

下面的变量用于向构建系统描述如可构建一个模块，每个模块都应遵守如下流程：

1. 使用 CLEAR_VARS 变量清理与上一个模块相关的变量；
2. 为用于描述模块的变量赋值；
3. 包含 BUILD_XXX 变量以适当的构建脚本用于该模块的构建。

- LOCAL_PATH

用于指定当前文件的路径，必须在 Android.mk 文件开头定义此变量。

`CLEAR_VARS` 指向的脚本不会清除此变量。

```makefile
# my-dir 是一个宏函数，返回当前 Android.mk 文件路径
LOCAL_PATH := $(call my-dir)
```

- LOCAL_MODULE

用于向构建系统描述模块名称，对于 .so 和 .a 文件，系统会自动给名称添加 `lib` 前缀和文件扩展名。

```makefile
# 产出 libfoo.so 或 libfoo.a
LOCAL_MODULE := foo
```

- LOCAL_MODULE_FILENAME

向构建系统描述模块的自定义名称，覆盖 `LOCAL_MODULE` 的名称。

```makefile
LOCAL_MODULE := foo
# 产出 libnewfoo.so，但无法改变扩展名
LOCAL_MODULE_FILENAME := libnewfoo
```

- LOCAL_SRC_FILES

向构建系统描述生成模块时所用的源文件列表，务必使用 Unix 样式的正斜杠 (/) 来描述路径，且避免使用绝对路径。

- LOCAL_CPP_EXTENSION

为 C++ 源文件指定除 .cpp 外的扩展名。

```makefile
LOCAL_CPP_EXTENSION := .cxx
```

或指定多个：

```makefile
LOCAL_CPP_EXTENSION := .cxx .cpp .cc
```

- LOCAL_CPP_FEATURES

向构建系统指明代码所依赖于的特定 C++ 功能。避免使用 `LOCAL_CPPFLAGS` 声明，它会导致编译器将所有指定的标记用于所有模块。

```makefile
# 使用运行时信息
LOCAL_CPP_FEATURES := rtti
```



```makefile
# 使用 C++ 异常
LOCAL_CPP_FEATURES := exceptions
```

指定多个：

```makefile
LOCAL_CPP_FEATURES := rtti features
```

- LOCAL_C_INCLUDES

指定路径列表，以便在编译时添加到 include 搜索路径。搜索路径同时影响 ndk-gdb 调试路径。

```makefile
LOCAL_C_INCLUDES := sources/foo
```

通过 `LOCAL_CFLAGS` 或 `LOCAL_CPPFLAGS` 设置任何对应的包含标记前定义此变量。

- LOCAL_CFLAGS

构建 C 和 C++ 源文件时构建系统要传递的编译器标记，`LOCAL_CPPFLAGS` 可仅为 C++ 源文件指定标记。

相关：GCC 编译器选项参考 [https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#C_002b_002b-Dialect-Options](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#C_002b_002b-Dialect-Options)

```makefile
# 指定额外 include 路径，推荐用 LOCAL_C_INCLUDES
LOCAL_CFLAGS += -I<path>,
```

- LOCAL_CPPFLAGS

只构建 C++ 源文件传递的一组编译器标记，放在 `LOCAL_CFLAGS` 变量定义的后面。

- LOCAL_STATIC_LIBRARIES

存储当前模块依赖的静态库模块列表

1. 如果当前模块是共享库或可执行文件，此变量强制这些库链接到生成的二进制文件；
2. 如果当前模块是静态库，此变量指出依赖于当前模块的其他模块也会依赖于其列出的库。

- LOCAL_SHARED_LIBRARIES

此变量列出此模块在运行时依赖的共享库模块。用于将相应的连链接信息嵌入到生成的文件中。

- LOCAL_WHOLE_STATIC_LIBRARIES

`LOCAL_STATIC_LIBRARIES` 的变体形式，表示链接器应将相关的库模块视为完整归档（链接所有符号，而不只是用到的），
可参考 ld 链接器的 `--whole-archive` 选项。

- LOCAL_LDLIBS

列出在构建共享库或可执行文件时使用的额外链接器标记，使用 `-l` 前缀来指明连接到特定系统库（一般用于链接 NDK 提供的公开系统库，例如 liblog）。

```makefile
# 链接 /system/lib/libz.so 模块
LOCAL_LDLIBS := -lz
```

- LOCAL_LDFLAGS

列出构建系统在构建共享库或可执行文件时使用的其他链接器标记。

```makefile
# 在 ARM/X86 上使用 ld.bfd 链接器
LOCAL_LDFLAGS += -fuse-ld=bfd
```

定义静态库时，构建系统会忽略此变量，`ndk-build` 会打印警告。

- LOCAL_ALLOW_UNDEFINED_SYMBOLS

默认情况下，构建系统在尝试构建共享库时遇到未定义的引用，将会抛出“未定义的符号”错误，指定此变量为 `true`，将停用此检查（可能会导致运行时加载）。

定义静态库时，构建系统会忽略此变量，`ndk-build` 会打印警告。

- LOCAL_ARM_MODE

默认情况下，构建系统会以 thumb 模式生成 ARM 目标二进制文件，其中每条指令都是 16 位宽，并与 thumb/ 目录中的 STL 库链接。将此变量定义为 arm 会强制构建系统以 32 位 arm 模式生成模块的对象文件。

```makefile
LOCAL_ARM_MODE := arm
```

或者对源文件名附加 .arm 后缀，指示构建系统仅以 arm 模式构建特定的源文件。

```makefile
# 以 ARM 模式编译 bar.c，但根据 LOCAL_ARM_MODE 的值构建 foo.c
LOCAL_SRC_FILES := foo.c bar.c.arm
```

也可以在 Application.mk 文件中将 APP_OPTIM 设置为 debug，强制构建系统生成 ARM 二进制文件。指定 debug 会强制构建 ARM，因为工具链调试程序无法正确处理 Thumb 代码。

- LOCAL_ARM_NEON

此变量仅在以 armeabi-v7a ABI 为目标时才有意义。它允许在 C 和 C++ 源文件中使用 ARM Advanced SIMD (NEON) 编译器固有特性，以及在 Assembly 文件中使用 NEON 指令

并非所有基于 ARMv7 的 CPU 都支持 NEON 扩展指令集。因此，必须执行运行时检测，以便在运行时安全地使用此代码。

```makefile
# 以 Thumb 和 NEON 支持编译 foo.c，以 Thumb 支持编译 bar.c，并以 ARM 和 NEON 支持编译 zoo.c
LOCAL_SRC_FILES = foo.c.neon bar.c zoo.c.arm.neon
```

同时使用这两个后缀时，`.arm` 必须在 `.neon` 前面。

- LOCAL_DISABLE_FORMAT_STRING_CHECKS

默认情况下，构建系统会在编译代码时保护格式字符串。这样的话，如果 `printf` 样式的函数中使用了非常量格式的字符串，就会强制引发编译器错误。

可通过将此变量的值设置为 true 将其停用，不建议停用。

- LOCAL_EXPORT_CFLAGS

记录一组 C/C++ 编译器标记，这些标记将被添加到使用通过 LOCAL_STATIC_LIBRARIES 或 LOCAL_SHARED_LIBRARIES 变量所描述模块的其他模块的 LOCAL_CFLAGS 定义中。

如下，foo 模块被 bar 模块依赖，那么标记 `-DFOO=1` 将在 bar 模块构建时和 `-DBAR=2` 一起传递至编译器。

```makefile
include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := foo/foo.c
LOCAL_EXPORT_CFLAGS := -DFOO=1
include $(BUILD_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := bar
LOCAL_SRC_FILES := bar.c
LOCAL_CFLAGS := -DBAR=2
LOCAL_STATIC_LIBRARIES := foo
include $(BUILD_SHARED_LIBRARY)
```

构建系统单独编译 foo 模块时，不会将 -DFoo 标记传递至编译器。

如果有其他模块例如 zoo 依赖于 bar，那么标记将被传递。

- LOCAL_EXPORT_CPPFLAGS

与 `LOCAL_EXPORT_CFLAGS` 相同，但仅适用于 C++ 标记。

- LOCAL_EXPORT_C_INCLUDES

与 `LOCAL_EXPORT_CFLAGS` 相同，但适用于 C include 路径。

- LOCAL_EXPORT_LDFLAGS

与 `LOCAL_EXPORT_CFLAGS` 相同，但适用于链接器标记。

- LOCAL_EXPORT_LDLIBS

此变量与 `LOCAL_EXPORT_CFLAGS` 相同，用于指示构建系统将特定系统库的名称传递到编译器。请在您指定的每个库名称前附加 -l

构建系统会将导入的链接器标记附加到模块的 `LOCAL_LDLIBS` 变量值上。其原因在于 Unix 链接器的工作方式

对于静态库会很有用：

```makefile
include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := foo/foo.c
LOCAL_EXPORT_LDLIBS := -llog
include $(BUILD_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := bar
LOCAL_SRC_FILES := bar.c
LOCAL_STATIC_LIBRARIES := foo
include $(BUILD_SHARED_LIBRARY)
```

那么构建系统在构建 libbar.so 时，将在链接器命令的末尾指定 -llog。告知链接器，由于 libbar.so 依赖于 foo，所以它也依赖于系统日志记录库。

- LOCAL_SHORT_COMMANDS

当模块有很多源文件和/或依赖的静态或共享库时，请将此变量设置为 `true`，这样会强制构建系统将 `@` 语法用于包含中间对象文件或链接库的归档。

此功能在 Windows 上可能很有用，在 Windows 上，命令行最多只接受 8191 个字符，这对于复杂的项目来说可能太少。它还会影响个别源文件的编译，而且将几乎所有编译器标记都放在列表文件内。

此功能会减慢构建速度。

- LOCAL_THIN_ARCHIVE

构建静态库时，请设置为 `true`。这样会生成一个瘦归档，即一个库文件，其中不含对象文件，而只包含它通常包含的实际对象的文件路径。

在非静态库模块或预构建的静态库模块中，将会忽略此变量。

- LOCAL_FILTER_ASM

请将此变量定义为一个 shell 命令，供构建系统用于过滤根据您为 `LOCAL_SRC_FILES` 指定的文件提取或生成的汇编文件。定义此变量会导致发生以下情况：

1. 构建系统从任何 C 或 C++ 源文件生成临时汇编文件，而不是将它们编译到对象文件中；
2. 构建系统在任何临时汇编文件以及 `LOCAL_SRC_FILES` 中所列任何汇编文件的 `LOCAL_FILTER_ASM` 中执行 shell 命令，因此会生成另一个临时汇编文件；
3. 构建系统将这些过滤的汇编文件编译到对象文件中。

```makefile
LOCAL_SRC_FILES  := foo.c bar.S
LOCAL_FILTER_ASM :=

foo.c --1--> $OBJS_DIR/foo.S.original --2--> $OBJS_DIR/foo.S --3--> $OBJS_DIR/foo.o
bar.S
```

“1”对应于编译器，“2”对应于过滤器，“3”对应于汇编程序。过滤器必须是一个独立的 shell 命令，它接受输入文件名作为第一个参数，接受输出文件名作为第二个参数。例如：

```
myasmfilter $OBJS_DIR/foo.S.original $OBJS_DIR/foo.S
myasmfilter bar.S $OBJS_DIR/bar.S
```



## NDK 提供的函数宏

NDK 提供了一些 GNU Make 的函数宏，使用 `$(call <function>)` 调用求值，返回相应文本信息。

- my-dir

返回最后包括的 makefile 的路径，通常是当前 Android.mk 的目录。

由于 GNU Make 的工作方式，这个宏实际返回的是构建系统解析构建脚本时包含的最后一个 makefile 的路径。因此，包括其他文件后就不应调用 my-dir，可以提前把返回值保存起来，避免受影响。

```makefile
MY_LOCAL_PATH := $(call my-dir)

LOCAL_PATH := $(MY_LOCAL_PATH)

# ... declare one module

include $(LOCAL_PATH)/foo/`Android.mk`

LOCAL_PATH := $(MY_LOCAL_PATH)

# ... declare another module
```

- all-subdir-makefiles

返回位于当前 my-dir 路径所有子目录中的 Android.mk 文件列表

利用此函数，您可以为构建系统提供深度嵌套的源目录层次结构。默认情况下，NDK 只在 Android.mk 文件所在的目录中查找文件。

- this-makefile

返回当前 makefile（构建系统从中调用函数）的路径。

- parent-makefile

返回包含树中父 makefile 的路径（包含当前 makefile 的 makefile 的路径）。

- grand-parent-makefile

返回包含树中祖父 makefile 的路径（包含当前父 makefile 的 makefile 的路径）。

- import-module

此函数用于按模块名称来查找和包含模块的 Android.mk 文件：

```makefile
$(call import-module,<name>)
```

构建系统在 `NDK_MODULE_PATH` 环境变量所引用的目录列表中查找具有 `<name>` 标记的模块，并且自动包括其 Android.mk 文件



# Application.mk 变量参考

Application.mk 指定 NDK 工程的项目级设置。

许多参数具有模块等效项，例如，`APP_CFLAGS` 对应于 `LOCAL_CFLAGS`，基于特定模块的选项优于项目级的选项。

对于标记来说，如果两者都使用，那么特定于模块的标记将后出现在命令行中，因此它们会替换项目级设置。

- APP_ABI

默认情况下，NDK 构建系统会为所有有效的 ABI 生成代码。可以使用 APP_ABI 设置为特定 ABI 生成代码。

| Instruction set        | Value                  |
| ---------------------- | ---------------------- |
| 32-bit ARMv7           | APP_ABI := armeabi-v7a |
| 64-bit ARMv8 (AArch64) | APP_ABI := arm64-v8a   |
| x86                    | APP_ABI := X86         |
| x86-64                 | APP_ABI := x86_64      |
| All supported ABIs (default) | APP_ABI：= all   |

可指定多个值：

```makefile
APP_ABI := armeabi-v7a arm64-v8a x86
```

Gradle 中的 `externalNativeBuild` 设置会忽略 `APP_ABI`。需要在 `splits` 块内部使用 `abiFilters` 块或 `abi` 块。

- APP_ASFLAGS

要传递给项目中每个汇编源文件（.s 和 .S 文件）的编译器的标记。

`ASFLAGS` 与 `ASMFLAGS` 不同。后者专用于 YASM 源文件。

APP_BUILD_SCRIPT

如需从其他位置加载 Android.mk 文件，将 `APP_BUILD_SCRIPT` 设置为 Android.mk 文件的绝对路径。

Gradle 中的 `externalNativeBuild` 块将根据 `externalNativeBuild.ndkBuild.path` 变量自动设置此路径。

- APP_CFLAGS

为项目中的所有 C/C++ 编译传递的标记。

- APP_CLANG_TIDY

为项目中的所有模块启用 clang-tidy，将此标记设置为 `True`。默认为停用状态。

- APP_CLANG_TIDY_FLAGS

要为项目中的所有 clang-tidy 执行传递的标记。

- APP_CONLYFLAGS

要为项目中的所有 C 编译传递的标记。这些标记不会用于 C++ 代码。

- APP_CPPFLAGS

要为项目中的所有 C++ 编译传递的标记。这些标记不会用于 C 代码。

- APP_CXXFLAGS

`APP_CPPFLAGS` 应优先于 `APP_CXXFLAGS`。

与 `APP_CPPFLAGS` 相同，但在编译命令中将出现在 `APP_CPPFLAGS` 之后。例如：

```makefile
APP_CPPFLAGS := -DFOO
APP_CXXFLAGS := -DBAR
```

以上配置将导致编译命令类似于 `clang++ -DFOO -DBAR`，而不是 `clang++ -DBAR -DFOO`。

- APP_DEBUG

构建可调试的应用，将此标记设置为 `True`。

- APP_LDFLAGS

关联可执行文件和共享库时要传递的标记。

这些标记对静态库没有影响。不会关联静态库。

- APP_MANIFEST

AndroidManifest.xml 文件的绝对路径。

默认情况下将使用 $(APP_PROJECT_PATH)/AndroidManifest.xml)（如果存在）。

使用 `externalNativeBuild` 时，Gradle 不会设置此值。

- APP_MODULES

要构建的模块的显式列表。此列表的元素是模块在 Android.mk 文件的 `LOCAL_MODULE` 中显示的名称。

默认情况下，ndk-build 将构建所有共享库、可执行文件及其依赖项。仅当项目使用静态库、项目仅包含静态库或者在 `APP_MODULES` 中指定了静态库时，才会构建静态库。

不会构建导入的模块（在使用 $(call import-module) 导入的构建脚本中定义的模块），除非要在 APP_MODULES 中构建或列出的模块依赖导入的模块。

- APP_OPTIM

定义为 `release` 或 `debug`。默认情况下，将构建 `relase` 模式的二进制文件。

`release` 模式会启用优化，并可能生成无法与调试程序一起使用的二进制文件。`debug` 模式会停用优化，以便可以使用调试程序。

应用清单的 `<application>` 标记中声明 `android:debuggable` 将导致此变量默认为 `debug`，而不是 `release`。将 `APP_OPTIM` 设置为 `release` 可替换此默认值。

使用 externalNativeBuild 进行构建时，Android Studio 将根据您的构建风格适当地设置此标记。

- APP_PLATFORM

声明构建此应用所面向的 Android API 级别，并对应于应用的 `minSdkVersion`。

如果未指定，ndk-build 将以 NDK 支持的最低 API 级别为目标。最新 NDK 支持的最低 API 级别总是足够低，支持几乎所有有效设备。

将 `APP_PLATFORM` 设置为高于应用的 `minSdkVersion` 可能会生成一个无法在旧设备上运行的应用。在大多数情况下，库将无法加载，因为它们引用了在旧设备上不可用的符号。

使用 Gradle 和 `externalNativeBuild` 时，不应直接设置此参数。而应在模块级别 build.gradle 文件的 `defaultConfig` 或 `productFlavors` 块中设置 `minSdkVersion` 属性。这样就能确保只有在运行足够高 Android 版本的设备上安装的应用才能使用您的库。

NDK 不包含 Android 每个 API 级别的库，省略了不包含新的原生 API 的版本以节省 NDK 中的空间。ndk-build 按以下优先级降序使用 API：

1. 匹配 `APP_PLATFORM` 的平台版本。
2. 低于 `APP_PLATFORM` 的下一个可用 API 级别。例如，`APP_PLATFORM` 为 `android-20` 时，将使用 `android-19`，因为 `android-20` 中没有新的原生 API;
3. NDK 支持的最低 API 级别。

- APP_PROJECT_PATH

项目根目录的绝对路径。

- APP_SHORT_COMMANDS

`LOCAL_SHORT_COMMANDS` 的项目级等效项。

- APP_STL

用于此应用的 C++ 标准库。

默认情况下使用 `system STL`。其他选项包括 `c++_shared`、`c++_static` 和 `none`。

- APP_STRIP_MODE

要为此应用中的模块传递给 `strip` 的参数。默认为 `--strip-unneeded`。若要避免剥离模块中的所有二进制文件，请将其设置为 `none`。

- APP_THIN_ARCHIVE

为项目中的所有静态库使用瘦归档，将此变量设置为 `True`。

- APP_WRAP_SH

要包含在此应用中的 `wrap.sh` 文件的路径。

每个 ABI 都存在此变量的变体，ABI 通用变体也是如此：

```
APP_WRAP_SH
APP_WRAP_SH_armeabi-v7a
APP_WRAP_SH_arm64-v8a
APP_WRAP_SH_x86
APP_WRAP_SH_x86_64
```

`APP_WRAP_SH_<abi>` 可能无法与 `APP_WRAP_SH` 结合使用。如果有任何 ABI 使用特定于 ABI 的 `wrap.sh`，所有 ABI 都必须使用该 `wrap.sh`。



# NDK API

NDK 开发几乎必须要使用到 NDK 提供的原生 API，最常用的就是 `liblog`，用来在 logcat 中打印日志，下面分别使用 Android.mk 和 CMake 引入日志库。

引入其他库方法一致，可用 NDK 库列表可参考官方文档：[https://developer.android.google.cn/ndk/guides/stable_apis](https://developer.android.google.cn/ndk/guides/stable_apis)



## Android.mk

非常简单，只需要在 Android.mk 文件中使用 `LOCAL_LDLIBS` 变量使用 `-l` 前缀描述需要连接的库即可：

```makefile
# Android.mk

LOCAL_PATH := $(call my-dir)

$(warning $(TARGET_PLATFORM))

include $(CLEAR_VARS)

LOCAL_MODULE := foo
LOCAL_SRC_FILES := libfoo.cpp

# 添加日志库，需要添加其他库可直接使用空格分隔
LOCAL_LDLIBS := -llog

include $(BUILD_SHARED_LIBRARY)
```

此时在源代码中即可使用 `android/log.h` 引入日志打印方法了。

```c++
// libfoo.cpp

#include <android/log.h>
#include "main.h"
#include <android/log.h>

static const char *TAG = "NDK";

extern "C" {

jstring Java_io_l0neman_mkexample_NativeHandler_getHello(JNIEnv *env, jclass clazz) {
  __android_log_print(ANDROID_LOG_DEBUG, TAG, "log test.");
  return env->NewStringUTF("Hello-jni");
}

};
```



## CMake

CMake 描述如下，首先使用 `find_library` 描述 NDK 库，再用 `target_link_libraries` 指定链接库：

```cmake
# CMakeLists.txt

cmake_minimum_required(VERSION 3.4.3)

add_library(
        foo
        SHARED
        main.cpp
)

find_library(
        # 使用变量描述系统库
        log-lib
        # 系统库名字
        log
)

# 指定将前面描述的 log-lib 库链接到目标 foo 中
target_link_libraries(
        foo
        ${log-lib}
)
```

如果需要添加多个库，新增 `find_library` 块，添加另一个库的描述后，在 `target_link_libraries` 加入即可：

```cmake
# CMakeLists.txt

...

find_library(
        zip-lib
        z
)

target_link_libraries(
        foo
        ${log-lib}
        ${zip-lib}
)
```



# 引入预编译库

有时需要引入提前编译好或者第三方提供的 so 共享库，或是引入现成的 .a 静态库，那么根据情况进行如下配置。



## 引入动态库

1. 首先在独立的 NDK 工程编译出一个共享库 libbar.so（创建 libbar Module），作为第三方库提供给其他 Module 使用。

工程目录结构：

```
jni
 |
 +-- Android.mk
 +-- Application.mk
 +-- libbar.h
 +-- libbar.cpp
```

测试代码：

```cpp
// libbar.h
#ifndef NDKTPROJECT_LIBBAR_H
#define NDKTPROJECT_LIBBAR_H

extern "C" {

int bar_add(int a, int b);

};

#endif //NDKTPROJECT_LIBBAR_H
```



```cpp
// libbar.cpp
#include "libbar.h"

int bar_add(int a, int b) {
  return a + b;
}
```



```makefile
# libbar/src/main/jni/Android.mk

LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := bar
LOCAL_SRC_FILES := libbar.cpp

include $(BUILD_SHARED_LIBRARY)
```



```makefile
# libbar/src/main/jni/Application.mk

APP_ABI := armeabi-v7a arm64-v8a x86 x86_64
APP_OPTIM := debug
```

使用命令行进入 jni 目录下，然后执行 ndk-build 编译出 4 种架构的 libbar.so 文件，在和 jni 同级的 libs 目录下。

```
jni
libs
 |
 +-- armeabi-v7a
 |    +-- libbar.so
 |
 +-- arm64-v8a
 |    +-- libbar.so
 |
 +-- x86
 |    +-- libbar.so
 |
 +-- x86_64
      +-- libbar.so
```



2. 将每种架构目录复制到需要使用此库的 NDK 工程中（libfoo.so Module），在工程中新建 include 目录，将 libbar 的头文件复制过来，为了提供调用的接口。

工程目录结构：

```
jni
 |
 +-- armeabi-v7a
 |    +-- libbar.so
 |
 +-- arm64-v8a
 |    +-- libbar.so
 |
 +-- x86
 |    +-- libbar.so
 |
 +-- x86_64
 |    +-- libbar.so
 |
 +-- include
 |    +-- libbar.h
 |
 +-- Android.mk
 +-- Application.mk
 +-- libfoo.h
 +-- libfoo.cpp
```



3. 编写 libfoo.so Module 的 Android.mk 文件，`$(TARGET_ARCH_ABI)` 为 NDK 编译时每种架构的名字。

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
# 描述预编译库动态库的名称
LOCAL_MODULE := libbar-pre
# 描述预编译动态库路径
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libbar.so
# 描述预编译动态库引入的头文件
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/include
include $(PREBUILT_SHARED_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := main.cpp
# 描述要使用的共享库名称
LOCAL_SHARED_LIBRARIES := libbar-pre
include $(BUILD_SHARED_LIBRARY)
```

此时当工程编译时，对应的 libbar.so 将会自动被加入到 apk 包中。



4. 代码调用

```cpp
// libfoo.h

#ifndef NDKTPROJECT_LIBFOO_H
#define NDKTPROJECT_LIBFOO_H

#include <jni.h>

extern "C" {

JNIEXPORT void JNICALL
Java_io_l0neman_mkexample_NativeHandler_test(JNIEnv *env, jclass clazz);

};

#endif //NDKTPROJECT_LIBFOO_H
```



```cpp
// libfoo.cpp

#include "libbar.h"
#include "libfoo.h"

void Java_io_l0neman_mkexample_NativeHandler_test(JNIEnv *env, jclass clazz) {
  int a = bar_add(1, 4);
  printf("%d\n", a);
}
```



5. Java 层调用测试

```java
// class io.l0neman.mkexample.NativeHandler

public class NativeHandler {

  static {
    // 加载 libfoo.so 时，libbar 会被自动加载。
    System.loadLibrary("foo");
  }

  public static native void test();
}
```



```java
// MainActivity.java
NativeHandler.test();
```



## 引入静态库

1. 首先编译出 .a 后缀的静态库 libbar.a。

工程结构和上面引入动态库中的 libbar 工程一致，只需要将 Android.mk 文件中引入的 `BUILD_SHARED_LIBRARY` 变量修改为 `BUILD_STATIC_LIBRARY` 即可指定编译出静态库。

```makefile
# libbar/src/main/Android.mk

LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := bar
LOCAL_SRC_FILES := libbar.cpp

# 指定编译出静态库
include $(BUILD_STATIC_LIBRARY)
```

使用 ndk-build 编译后，不会产生和 jni 同级的 libs 目录，每种架构的 libbar.a 文件将出现在和 jni 同级的 obj 目录中。

目录结构如下：

```
jni
obj
 |
 +-- armeabi-v7a
 |    +-- libbar.a
 |
 +-- arm64-v8a
 |    +-- libbar.a
 |
 +-- x86
 |    +-- libbar.a
 |
 +-- x86_64
      +-- libbar.a
```



2. 在 libfoo.so 工程中引入静态库，步骤和引入动态库大同小异，把 obj 中每种架构的目录复制到需要使用此库的 NDK 工程中（libfoo.so），在工程中新建 include 目录，将 libbar 的头文件复制过来，为了提供调用的接口。

工程目录结构：

```
jni
 |
 +-- armeabi-v7a
 |    +-- libbar.a
 |
 +-- arm64-v8a
 |    +-- libbar.a
 |
 +-- x86
 |    +-- libbar.a
 |
 +-- x86_64
 |    +-- libbar.a
 |
 +-- include
 |   +-- libbar.h
 |
 +-- Android.mk
 +-- Application.mk
 +-- libfoo.h
 +-- libfoo.cpp
```



3. 编写 libfoo.so 的 Android.mk 文件，`$(TARGET_ARCH_ABI)` 为 NDK 编译时每种架构的名字。

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
# 描述预编译静态库的名字
LOCAL_MODULE := libbar-pre
# 描述预编译静态库的位置
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libbar.a
# 描述预编译静态库引入的头文件
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/include
include $(PREBUILT_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := main.cpp
# 描述要使用的静态库名称
LOCAL_STATIC_LIBRARIES := libbar-pre
include $(BUILD_SHARED_LIBRARY)
```

此时当工程编译时，对应的 libbar.a 将会自动编译到 libfoo.so 中，成为它的一部分。



4. 最后引用头文件正常调用编译即可，参考引用动态库中的步骤 4。


## CMake

上面的两个示例均为 Android.mk 构建示例，使用 CMake 构建简要描述如下：

- 引入动态库

首先将前面 libbar.so 复制到 CMake 项目的 jniLibs 中，项目结构如下：

```
main
 |
 +-- cpp
 |    |
 |    +-- include
 |    |    |
 |    |    +-- libbar.h
 |    |
 |    +-- libfoo.cpp
 |    +-- libfoo.h
 |    +-- CMakeLists.txt
 |
 +-- jniLibs
      |
      +-- armeabi-v7a
      |    +-- libbar.so
      |
      +-- arm64-v8a
      |    +-- libbar.so
      |
      +-- x86
      |    +-- libbar.so
      |
      +-- x86_64
           +-- libbar.so
```

将预编译库放在 jniLibs 下面是为了在编译时打包到 apk 中。

其中 libfoo.cpp 和 libfoo.h 与上述 Android.mk 中源码一致，重点关注 CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.4.3)
# 设置当前路径变量
set(CURRENT_DIR ${CMAKE_SOURCE_DIR})

add_library(
        foo
        SHARED
        main.cpp
)
# 描述预编译动态库 bar-lib
add_library(
        bar-lib
        SHARED
        IMPORTED
)
# 设置预编译库 bar-lib 位置属性
set_target_properties(
        bar-lib
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libbar.so
)
# 为上面预编译库指定头文件路径
include_directories(include/)

target_link_libraries(
        foo
        bar-lib
)
```

编译测试即可。



- 引入静态库

将前面 libbar.a 复制到 CMake 项目的 cpp 中，项目结构如下：

```
cpp
 |
 +-- include
 |    |
 |    +-- libbar.h
 |
 +-- libfoo.cpp
 +-- libfoo.h
 +-- CMakeLists.txt
 |
 +-- armeabi-v7a
 |    +-- libbar.a
 |
 +-- arm64-v8a
 |    +-- libbar.a
 |
 +-- x86
 |    +-- libbar.a
 |
 +-- x86_64
      +-- libbar.a
```

由于静态库 .a 直接编译到目标文件 libfoo 中，所以不用放在 jniLibs 打包至 apk 中。

CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.4.3)
# 设置当前路径变量
set(CURRENT_DIR ${CMAKE_SOURCE_DIR})

add_library(
        foo
        SHARED
        main.cpp
)
# 描述预编译库静态库 bar-lib
add_library(
        bar-lib
        STATIC
        IMPORTED
)
# 设置预编译库 bar-lib 位置属性
set_target_properties(
        bar-lib
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/${ANDROID_ABI}/libbar.a
)
# 为上面预编译库指定头文件路径
include_directories(include/)

target_link_libraries(
        foo
        bar-lib
)
```

编译测试即可。



# 参考

[https://developer.android.google.cn/ndk/guides/build](https://developer.android.google.cn/ndk/guides/build)
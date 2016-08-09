title: 将 ffmpeg 编译为 android JNI 库
tags:
  - Android
  - ffmpeg
categories:
  - Programmer
date: 2016-07-27 15:14:38
---
之前有开源了一个 [ffmpeg shared library](https://github.com/dxjia/ffmpeg-compile-shared-library-for-android)，其中是把 ffmpeg 2.6 源码编译为android 平台可用的共享库，在其中只介绍了如何编译，但对于如何将这些共享库转变为 JNI 使用却没有讲太多。本文就着重介绍一下思路，方便后续升级 ffmpeg 版本。

<!--more-->

# 编译 ffmpeg 共享库

依据 [使用](https://github.com/dxjia/ffmpeg-compile-shared-library-for-android#使用) 介绍下载 ffmpeg 源码，配置编译脚本进行编译。
`build_android_arm.sh` 和 `build_android_x86.sh` 两个脚本分别是用来编译用于 arm 平台的库文件和 x86 平台的库的，如果你只需要 arm 平台的，那 x86 的可以不用管了。
**`注意：`每次运行完脚本编译之后，会在 ffmpeg 源码目录下生成一个 config.h 文件，这个也是后续编译 jni 必需的，需要在运行另一个编译脚本之前进行保存， 我这里采用的方式是将 config.h 重命名为不同的名字，在编译 jni 时根据 target 不同引用不同的头文件，即：运行完 `build_android_arm.sh` 之后，将 config.h 重命名为 `arm_config.h`，运行完 `build_android_x86.sh`之后，重命名为 `x86_config.h`**

# 编译 jni

新建一个android项目，并新建一个 library module，也可以直接在你的 app 项目里，看自己的需要了。

## 禁用 android studio JNI 自动编译
在 你的 `build.gradle` 中增加下面几行代码，来禁用 as 对 jni 的自动编译，因为我们需要自己手动编译 jni。参考 [build.gradle](https://github.com/dxjia/ffmpeg-commands-executor-library/blob/as-version/library/build.gradle)。
```
    sourceSets.main {
        jni.srcDirs = [] //disable automatic ndk-build
        jniLibs.srcDirs = ['libs']
    }
```

## 准备文件

### Android.mk

在module下新建 jni 目录，在该目录下新建 `Android.mk` 文件，内容复制 [这里](https://github.com/dxjia/ffmpeg-commands-executor-library/blob/as-version/library/jni/Android.mk)，当然后续对该文件根据需要还有少许调整，后面会提到。

### 头文件

将从 ffmpeg 源码编译得到的 arm 和 x86 的 include 文件夹 整个复制到 jni 目录下，注意 arm和x86相同的文件直接覆盖或略过都可以的，主要是要让它们互相补充。

`后续实际编译你会发现，这些头文件并不够，还会缺少很多.h文件，所以后续随着编译的过程，必须自己一点点从ffmpeg源码目录不断补充头文件。`

### prebuilt

在 jni 目录下新建 `prebuilt` 目录，然后在 prebuilt 目录内，分别新建 `armeabi` 和 `x86` 目录，然后将 ffmpeg 源码编译的出来的 arm 的几个so库文件复制到 armeabi 目录下，同理，将 x86 的复制到 x86 目录下。
接下来，修改 Android.mk 文件中的对应的 so 名，都改成自己的对应文件名。如下：

```
nclude $(CLEAR_VARS)
LOCAL_MODULE:= avcodec-prebuilt-$(LIB_NAME_PLUS)
LOCAL_SRC_FILES:= prebuilt/$(LIB_NAME_PLUS)/libavcodec-56.so
include $(PREBUILT_SHARED_LIBRARY)
```

`不同的 ffmpeg 版本，这些so的版本是不同的，体现在so文件名都有个数字尾巴，这里不要去掉这些尾巴，因为它们之间还有依赖。如果确实不想带尾巴，那么就需要去改ffmpeg源码的makefile了`

### 复制源文件

`ffmpeg` 可执行文件执行命令的入口是 ffmpeg.c 文件的 main 函数，所以我们的原理就是将个这个main函数修改为我们的jni函数，那么java层调用这个jni接口就可以执行命令了。
从ffmpeg 源码目录下的复制下面的文件到jni目录下

```
ffmpeg.c
ffmpeg.h
arm_config.h
x86_config.h
cmdutils.c
cmdutils.h
cmdutils_common_opts.h
ffmpeg_filter.c
ffmpeg_opt.c
```

调整 `Android.mk` 的 `LOCAL_SRC_FILES` 部分，将 jni 目录下的 所有 .c 文件都加入其中，这样才能参与编译。

### 新建 config.h 文件
内容如下：

```
#if USE_ARM_CONFIG
#include "arm_config.h"
#elif USE_X86_CONFIG
#include "x86_config.h"
#endif
```

### 增加 jni 接口
如何增加jni这里不多讲了，网上资料很多，也可以参考 [library](https://github.com/dxjia/ffmpeg-commands-executor-library/tree/as-version/library).
将 ffmpeg.c main函数修改为你的 jni接口，然后在 jni 目录下执行 `build.sh` (windows下执行 build.cmd) 进行手动编译，一般这里你就会发现少很多头文件，可以一点点试，缺少哪个头文件就从ffmpeg源码目录copy到include的对应目录下，注意有些是需要保留目录结构的，没有就新建。

### java 部分
在java部分的jni文件中，使用static代码块加载lib库，如下：

```
	static {
		System.loadLibrary("avutil-54");
		System.loadLibrary("swresample-1");
		System.loadLibrary("avcodec-56");
		System.loadLibrary("avformat-56");
		System.loadLibrary("swscale-3");
		System.loadLibrary("avfilter-5");
		System.loadLibrary("avdevice-56");
		System.loadLibrary("ffmpegjni");
	}
```

`注意名字，修改为你实际使用的版本，并且名字上不带lib`，参考 [FFmpegNativeHelper.java](https://github.com/dxjia/ffmpeg-commands-executor-library/blob/as-version/library/src/main/java/cn/dxjia/ffmpeg/library/FFmpegNativeHelper.java),

至此，从 android studio里编译就可以将所有lib和jni库都编译到你的工程里去了。


`附：` 实际移植 ffmpeg3.0 的时候遇到的其他问题：
> 1. 在编译ffmpeg源码时，编译完 arm 的版本，去编 x86 版本的时候，提示 strtop 错误，这是由 ffmpeg 的makefile bug造成的，再 `run build_android_x86.sh` 之前先手动删除 `compat/strtop.o & compat/strtop.d` 文件。




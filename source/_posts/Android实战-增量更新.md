---
layout: post
title: Android实战-增量更新
mathjax: false
date: 2020-09-19 09:30:16
urlname: Android实战-增量更新
categories:
 - Android实战
tags:
 - Android实战
---

# Android实战-增量更新

增量更新是区别于全量更新而提出的。全量更新是指在应用更新时，每次都下载完整的安装包，进行覆盖安装。在Android早期，由于应用普遍比较小，因此为了简单便捷，普遍使用全量更新的方式，但是随着Android的发展，应用也变得越来越大，为了节省更新的时间，以及节省下载的流量，增量更新也就变得越来越普遍。

## 增量更新的原理

在发布新版本的apk时，使用差分算法，计算出与老版本apk的差异，两个版本之间的不同的部分，生成一个patch文件，在升级的时候，用户只需要下载这个patch文件，然后旧版本的app利用这个patch文件与自己的apk合成，从而得到新版本的apk文件，再进行安装，这样app就升级为新的版本了。

具体步骤一般是这样的：

1. 在发布新版本apk上传到服务器时，同时生成与以往每一个版本的patch文件，以便app端升级时使用。
2. app利用自己当前的版本信息向服务器查询升级服务，服务器一般会返回patch文件的下载地址，patch文件的MD5值，以及新版本APK的MD5值。
3. app下载patch文件，并校验patch文件的MD5值是否正确，如果准确无误，则利用patch文件与自己的apk文件进行合成，生成新的apk文件，然后校验生成的apk文件的MD5值是否正确，如果准确无误，则进行安装。

## 操作实践

增量升级的核心就是利用`diff/patch`算法来对新/旧apk进行`diff/patch`操作，当前主流的做法是利用`bsdiff`工具。

### 服务器端配置

首先需要在服务器端安装`bsdiff`工具，由于该工具依赖`bzip2`库，所以需要先安装`bzip2`库文件：

```bash
sudo apt-get install libbz2-dev #ubuntu 18.04
```

然后下载[bsdiff工具](https://github.com/mendsley/bsdiff/archive/v4.3.tar.gz)，当前版本为4.3，然后解压：

```bash
tar -zxvf bsdiff-4.3.tar.gz
cd bsdiff-4.3
```

这里连主要包含两个c文件：bsdiff.c用于生成patch补丁文件，bspatch.c用于将patch文件与旧版本apk合成。我们需要修改一下makefile文件：

```bash
CFLAGS          +=      -O3 -lbz2
CC = gcc

PREFIX          ?=      /usr/local
INSTALL_PROGRAM ?=      ${INSTALL} -c -s -m 555
INSTALL_MAN     ?=      ${INSTALL} -c -m 444

all:            bsdiff bspatch
bsdiff:         bsdiff.c
        $(CC) bsdiff.c $(CFLAGS) -o bsdiff
bspatch:        bspatch.c
        $(CC) bspatch.c $(CFLAGS) -o bspatch

install:
        ${INSTALL_PROGRAM} bsdiff bspatch ${PREFIX}/bin
        .ifndef WITHOUT_MAN
        ${INSTALL_MAN} bsdiff.1 bspatch.1 ${PREFIX}/man/man1
        .endif
```

然后我们执行`make`命令，这时候会在目录下生成`bsdiff`和`bspatch`两个可执行文件。生成patch文件可以使以下命令：

```bash
bsdiff oldFile newFile patchFile
```

这样服务器端，我们就得到了patch文件。

### 客户端配置

首先，我们先下载[`bzip2`源文件](ftp://sourceware.org/pub/bzip2/bzip2-1.0.6.tar.gz)并解压，然后在`Android Studio`中新建一个`C++ Native`工程，然后在`cpp`目录下，新建一个`bzip2`目录，然后将刚才`bzip2`解压后目录中的所有c文件和h文件拷贝到这个新建的目录中，然后将`bspatch.c`文件拷贝到`cpp`目录下，并删除自动生成的`native-lib.cpp`文件，最后我们修改一下`CMakeLists.txt`文件以及`bspatch.c`文件。

`CMakeLists.txt`文件：

```cmake
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.
             bspatch

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             bspatch.c )

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
                       bspatch

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

`bspatch.c`文件：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <err.h>
#include <unistd.h>
#include <fcntl.h>

#include <jni.h>
#include <sys/types.h>

#include "bzip2/bzlib.h"
#include "bzip2/bzlib.c"
#include "bzip2/crctable.c"
#include "bzip2/compress.c"
#include "bzip2/decompress.c"
#include "bzip2/randtable.c"
#include "bzip2/blocksort.c"
#include "bzip2/huffman.c"

JNIEXPORT jint JNICALL Java_com_example_smartupdate_MainActivity_patch(JNIEnv *env,
        jobject instance,
        jstring oldAPKPath_,
        jstring newAPKPath_,
        jstring patchPath_) {
    const char *oldAPKPath = (*env)->GetStringUTFChars(env, oldAPKPath_, 0);
    const char *newAPKPath = (*env)->GetStringUTFChars(env, newAPKPath_, 0);
    const char *patchPath = (*env)->GetStringUTFChars(env, patchPath_, 0);
    int argc = 4;
    char *argv[4];
    argv[0] = "bspatch";
    argv[1] = oldAPKPath;
    argv[2] = newAPKPath;
    argv[3] = patchPath;
    int ret = bspatch_main(argc, argv);
    (*env)->ReleaseStringUTFChars(env, oldAPKPath_, oldAPKPath);
    (*env)->ReleaseStringUTFChars(env, newAPKPath_, newAPKPath);
    (*env)->ReleaseStringUTFChars(env, patchPath_, patchPath);
    return ret;
}
//将原来的main函数，改名为bspatch_main函数，才能正常使用
```

这样编译之后，我们就可以得到`libbspatch.so`动态库了，增量更新有一个前提就是，必须有一个可供打patch的apk文件，我们可以在代码中来获取自己app的apk路径。

```kotlin
companion object {
        init {
            System.loadLibrary("bspatch")
        }
    }

external fun patch(oldAPKPath: String, newAPKPath: String, patchPath: String): Int

//获取自己app的apk文件路径
private fun getAPKPath():String?{
    val intent = Intent(Intent.ACTION_MAIN, null)
    intent.addCategory(Intent.CATEGORY_LAUNCHER)
    val infoList = packageManager.queryIntentActivities(intent, 0)
    var oldAPKPath:String? = null
    infoList.forEach { app ->
        val dir = app.activityInfo.applicationInfo.publicSourceDir
        if (dir.contains(packageName)) {
            oldAPKPath = dir
            return@forEach
        }
     }
    return oldAPKPath
}
```

这样，我们就可以调用`patch`方法生成新的apk文件(MD5值的校验本文暂不提及)，从而进行升级了。
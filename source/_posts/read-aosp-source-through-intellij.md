title: 使用Android Studio(Intellij)阅读AOSP源码
tags:
  - Android
  - Intellij
  - IDE
categories:
  - Programmer
date: 2016-03-18 14:27:47
---
`Android Studio`是Google推出的基于`Intellij`二次开发的IDE，用来替代之前的`eclipse+ADT`的开发方式，Studio针对Android开发提供了大量的优化，用起来非常方便，是开发APK的IDE不二之选。而本文来描述如何使用Android Studio或者Intellij来阅读Android的整个源码。
<!--more-->
之前一直是使用Source Insight来读源码的，这也是一个非常不错的Project Tool，但其是收费的，而且还挺贵（当然可能用破解的偏多，但如果在正规公司就不太好用破解啦。）。它的优势是比较轻量，速度还很快，占用空间也小，但需要你自己一点点区分module，它仅仅是作为一个代码阅读器，并不会像一个真正的IDE一样帮你区分模块，Android的模块又是那么的多，有时候经常并不清楚这个调用实现在哪个模块，只能自己去找，除非你把整个Android代码都加了进来，并全局搜索，而且很多时候关联跳转并不好。So，我们需要Android Studio....

进入正题：作为IDE很重要的一个功能是为你的project区分好模块，然后根据模块建立索引，而AOSP源码里就帮我们提供了这样的工具，我们要做的就是使用这个工具，生成Android Studio和Intellij可以使用的工程文件。

# 步骤

## 编译AOSP
完整编译一次AOSP，这样做的目的是为了生成`idegen`工具需要使用到的jar包 -- `idegen.jar`。
如果你`make`成功之后，发现在`/out/host/linux-x86/framework`目录下没有生成这个文件，那么可以自己手动编译，
```
 cd development/tools/idegen/
 mm
```

这之后就会生成`idegen.jar`。 

## 执行idegen.sh
在源码根目录执行：
> development/tools/idegen/idegen.sh

耐心等待，过一会就会在根目录下生成两个文件，`android.iml`和`android.ipr`，这就是我们需要的。

## 使用Intellij/Android Studio打开工程文件
打开Intellij和Android Studio，`File->open->android.ipr`，或者Andoird Stuido， `Open an existing Android Studio project -> android.ipr所在目录`, 之后就是漫长的等待，需要花很久的时间建立索引。

## 效果
图片用的别人的，原址不可靠了，在此感谢。
![](http://7xqitw.com1.z0.glb.clouddn.com/blog/res/android-stuido-read-aosp.png)

# idegen.jar
我将我编译出的`idegen.jar`放在了github上，地址: [read_aosp_through_intellij](https://github.com/dxjia/common-tools/tree/master/read_aosp_through_intellij)



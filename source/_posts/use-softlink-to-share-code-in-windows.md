title: 巧用软连接实现不同项目代码共享
tags:
  - Android
  - Windows
categories: Programmer
date: 2016-02-02 08:51:05
---
情形是这样的：
- 正在开发一个APP，这个APP向外提供一些服务，也就是一些数据接口，实现了数据的发送、接收，以及数据格式的定义和解析模块；
- 想在提供给使用者之前先自己测试一下数据的交互流程，采用的做法是自己再写一个测试APK，来模拟流程；
<!--more-->

废话了，其实总结来说，就是我有两个APK需要共用一部分核心代码。。。。

其实方法有好几种：
1. 把核心部分单独抽出来做成`library`，两个项目同时引用；-- 这不是我想要的，一来这个APP没有做库的需求，二来还要单独维护，jar包还要拷来拷去。。
2. 把测试APK作为一个module放到APP的项目里； -- 我也不想采用这种，尽管是一个module，但我每次发布APP代码，还要想着去删除，而且我也不需要让别人看到测试APK代码。

方法就剩下一个了，那就是让项目之间直接共用文件，玩过Linux的都知道，Linux家族里有个`ln`命令，可以用来给一个目录或文件创建一个`软链接`，访问这个链接时，就等同于访问实际的目录。。。。

那`windows`下有没有类似命令呢，答案是`有`。。`mklink`，使用它，我们可以在我们的测试APK代码目录下，将要使用到的APP那边的代码目录或文件映射过来，这样Android Studio就会看到文件，只有windows知道是软链接。。。。
映射目录：
```
mklink /j d:\目录 源目录
```
映射文件
```
mklink 文件名 源文件路径
```

不过这种方式，有个缺点，**那就是两边访问的代码是同一个，都是APP那边的，所以在测试APK里，只能使用，可不要修改这些文件哦。。。要不然，APP那边会容易出错的。。。**


另外，这种方式虽然看起来跟使用`快捷方式`差不多，但实际并不一样，测试发现，使用快捷方式，android studio是无法识别出来的。。。

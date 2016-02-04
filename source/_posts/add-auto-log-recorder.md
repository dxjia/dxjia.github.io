title: 为APP添加log自动记录功能
tags:
  - Android
  - Java
categories:
  - Programmer
date: 2016-02-03 15:01:00
---

在开发APP的时候很多情况下，在编码阶段，开发人员没有时间去做大量测试，主要测试工作还需要放在编码之后，在版本release之后由专业测试人员进行测试，而测试log的保存就变的尤为重要，本文就介绍如果在自己的APP中集成自动记录本APP的log到文件，以方便开发分析。
<!--more-->
> 本文最初实现参考自 <http://blog.csdn.net/way_ping_li/article/details/8487866> 这篇文章，对其进行了一些优化和扩展，比如增加保存的文件的大小控制，这样log文件达到一定的大小可以自动另起文件继续进行保存，防止文件太大，影响log分析。。

# 实现原理
Android手机log的抓取是通过一个logcat进程进行的，比如我们使用数据线连接真机抓取log时，会使用下面的命令进行，
```
adb logcat -v time
adb logcat -b radio -v time
```
`logcat`是手机中的一个可执行文件，所以我们可以利用新起一个线程，并在线程中通过
```
Runtime.getRuntime().exec(logcat command)
```
来运行这个logcat，并得到输出记录到文件。。。


# 主要实现
## 首先需要增加权限
在AndroidManifest文件中增加以下权限，
```
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_LOGS" />
```

## grep PID
自动记录log，我们只需要仅仅记录自己的APP产生的log即可，所以需要通过 grep PID的方式将其他log过滤掉。
```
    logcat | grep 
```

# 源码

源码放在 github上了 : <https://github.com/dxjia/LogRecorder>

# 使用方法
直接复制 [LogRecorder.java](https://github.com/dxjia/LogRecorder/blob/master/LogRecorder.java) 文件到你的工程中，使用方式如下：
首先在`AndroidManifest.xml`中增加权限：
```
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_LOGS" />
```

然后在代码中合适的地方使用：
```
	LogRecorder logRecorder
			 = new LogRecorder.Builder(context)
	               .setLogFolderName("foldername")
	               .setLogFolderPath("/sdcard/foldername")
	               .setLogFileNameSuffix("filesuffix")
	               .setLogFileSizeLimitation(256)
	               .setLogLevel(4)
	               .addLogFilterTag("ActivityManager")
	               .setPID(android.os.Process.myPid())
	               .build();

	logRecorder.start();
```

## setLogFolderName()
> 设定log输出目录名，如果该值与folder path都没有设定的话，会默认使用应用包名在sdcard下新建目录。

## setLogFolderPath()
> 设置Log输出目录绝对路径，该值会优先使用，会忽略folder name的设置。

## setLogFileNameSuffix()
> log文件名前缀，文件名会使用时间的形式，该值的设定会自动追加在时间之前，如setLogFileNameSuffix("mylog") 则最后的文件名为`mylog-2016-02-04-12-26-53.log`

## setLogFileSizeLimitation()
> 单个log文件的大小限制，超过设置的限制时，会自动新起新的文件记录log，**`注意`**: 是以`KB`为单位的。

## setLogLevel()
> 设置记录的log级别，默认2
> 2 = verbose 
> 3 = debug 
> 4 = info 
> 5 = warning 
> 6 = error 
> 7 = silent(不输出任何log)

## addLogFilterTag()
> 设置log过滤的tag，可以add多个，如`addLogFilterTag("ActivityManager")`表示只过滤“ActivityManager”的log 

## setPID()
> 通过该方法可以指定一个特定的进程的log，如通过`setPID(android.os.Process.myPid())` 即可只输出自己的APP的log。

# Reference
<http://blog.csdn.net/way_ping_li/article/details/8487866>

## 关于作者
- 微博：[@piggyguy](http://weibo.com/u/2139052944)
- 邮箱：<jdxwind@dxjia.cn>
- GitHub： [dxjia](https://github.com/dxjia)


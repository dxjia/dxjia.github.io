title: Android源码设置default application
tags:
  - Android
  - AOSP
categories:
  - Programmer
date: 2016-03-08 15:49:50
---
在做AOSP源码开发时，有时候为了OEM厂商，会将某些原生APP替换为厂商的APP，或者将厂商的APP设置为默认APP，本文来介绍如何在源码编译环境进行这样的功能设定。
<!--more-->
# 替换原生APP
比如原生有自带一个Calculator应用，OEM出ROM时希望直接替换掉原生应用，这个时候，我们可以将厂商的APP拿过来，放在vendors目录下，然后编写`makefile`文件，使用 `LOCAL_OVERRIDES_PACKAGES`就可以将ROM中删掉，自己取而代之，像下面这样写，后面跟要替换掉的 Module Name，这个name要去Calculator的`Android.mk`里去看。
```
    LOCAL_OVERRIDES_PACKAGES:= Calculator
```
# 保留原生，但设置自己的为开机默认
ROM中有多个可以提供相同功能的APP时，系统在用户使用时会提示用户进行选择，而且用户可以选择某个APP作为这项功能的默认应用，这样以后使用此项功能时，直接就启动默认APP，类似于我们在Windows上装了多个浏览器软件时，某些浏览器总是会提示`我现在不是你的默认浏览器，求你设置我为你的默认`， 噗～
那么如何在源码编译阶段就提前设置好呢，比如我们自己有一款浏览器APP，而且也不打算顶替掉原生的浏览器，但我们想提前随ROM一起设置自己为默认的浏览器软件。下面依此为例进行介绍：
## Andriod保存用户默认设置的地方
如果你了解过Android `PackageManager`的工作方式，你会知道，Android系统将系统中所有已经安装的APP信息都记录在一个xml文件里，路径为:
> /data/system/users/{*user-id*}/package-restrictions.xml

在这个文件里，详细记录了每个APP的各种组件信息，而APP默认设置的地方就保存在类似下面的块内容里：
```
    <preferred-activities>
    ...
    </preferred-activities>
```
`package-restrictions.xml`文件的生成是由`PackageManager`在开机阶段通过遍历 data/app， system/app, system/priv-app目录，并结合源码编译阶段的另一些配置文件来生成的，这些配置文件位于`/system/etc/preferred-activities/*.xml`目录，是编译阶段从源码中直接copy过来的，所以我们需要按格式要求书写我们的xml文件，并想办法让编译自动将xml文件复制到`system/etc/prefeered-activities/`目录下：

## 配置格式
比如上面提到的我们有一个 浏览器APP，需要提前设置好默认，那么在我们的AOSP源码的某个地方，我们定义一个xml文件，起名为` preferred-activies-mybrowser.xml`，内容如下:
```
<?xml version="1.0" encoding="UTF-8"?>
   <preferred-activities>
      <item name="com.mybrowser.MainActivity" match="200000" always="true" set="2">
         <set name="com.mybrowser./.MainActivity" />
         <set name="com.android.browser/.BrowserActivity" />
         <filter>
            <action name="android.intent.action.VIEW" />
            <cat name="android.intent.category.DEFAULT" />
            <scheme name="http" />
         </filter>
      </item>
   </preferred-activities>
```
## 复制xml到etc目录
这就需要项目配置makefile，在一个合适的地方加上下面的语句：
```
PRODUCT_COPY_FILES +=/<location-of-file>/preferred-activities-mybrowser.xml:system/etc/preferred-apps/preferred-activities-mybrowser.xml
```
这样编译的时候就可以自动复制xml文件啦。
## 缺点
该方式也有缺点：不能对Laucher APP这么干，使用上面的方式设置自己的Laucher应用为默认，会在第一次启动的时候失效。 


# Reference
[http://stackoverflow.com/questions/34073290/set-default-application-on-aosp](http://stackoverflow.com/questions/34073290/set-default-application-on-aosp)


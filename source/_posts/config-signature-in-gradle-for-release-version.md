title: 在gradle中为release版本配置签名
tags:
  - Android
  - APK
  - Gradle
categories:
  - Programmer
date: 2016-03-04 16:05:02
---
任何一个Android APK 发布之前都会进行签名，没有签名的APK是无法在Android device上进行安装和使用的，而且对于发布到Google Play上的同一个应用，自始至终必须使用同一个签名文件，所以必须保存好签名文件。本文介绍如何在`build.gradle`中为release版本配置签名文件，这样在打包release版本时可以自动进行签名。
<!--more-->

## 签名文件
签名APK首先需要一个签名文件，可以使用工具生成，或者使用Android Studio生成，Android使用的签名机制源自JAVA，签名文件可以使用JDK的工具进行生成：
```
keytool -genkey -alias dxjia -keyalg RSA -validity 20000 -keystore app.keystore
```
以管理员运行命令行，切换到JDK的bin目录下执行命令，否则`-keystore` 指定的文件名安排在其他非系统盘符目录，当然，如果你的JDK没有安装在系统盘，这个问题就不用担心。
![keytool command](http://7xqitw.com1.z0.glb.clouddn.com/keytool.png)

其中参数`-alias`后为别名，这个参数在签名APK时需要用到，最后一行输入的密钥口令是这个别名的密码，和一开始输入的密钥库口令，两个密码在签名时也会用到。`-validity`为证书有效天数，是的，证书有时间效力。在输入密码时没有回显(尽管输就是啦) 并且 退格,tab等都会被当作密码内容.

## 在gradle中进行配置
在android的gradle环境中，android提供了可以直接用于给release版本签名的变量，我们可以直接使用。
### 明文配置
```
...
android {
    ...
    defaultConfig { ... }
    signingConfigs {
        release {
            storeFile file("C\:\\Users\\.android\\app.keystore")
            storePassword "123456789"
            keyAlias "dxjia"
            keyPassword "987654321"
        }
    }
    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
        }
    }
}
...
```
这样在打包release版本时就会自动签名了。

### 隐藏密码
如果你的代码不会开源，或者是公司项目，公司会保护的很好，那么直接用上面的方式就可以了。但如果是开源项目，那么上面的写法就不太可取了 ，因为这样直接配置会暴露密码啊。接下来介绍如果隐藏密码进行配置：
> 利用`property`，将密码和签名文件路径信息保存在项目根目录的`local.properties`文件里，因为这个文件一般都在`.gitignore`里自动配置了，所以不会上传出去。

**在根目录的**`local.properties`文件中配置以下内容：
```
keystore.path=C\:\\Users\\.android\\app.keyset
keystore.password=123456789
keystore.alias=monkey
keystore.alias_password=987654321
```

然后在你的**app**目录的`build.gradle`文件中增加以下代码：
```
def keystoreFilepath = ''
def keystorePSW = ''
def keystoreAlias = ''
def keystoreAliasPSW = ''
// default keystore file, PLZ config file path in local.properties
def keyfile = file('s.keystore.temp')

Properties properties = new Properties()
// local.properties file in the root director
properties.load(project.rootProject.file('local.properties').newDataInputStream())
keystoreFilepath = properties.getProperty("keystore.path")

if (keystoreFilepath) {
    keystorePSW = properties.getProperty("keystore.password")
    keystoreAlias = properties.getProperty("keystore.alias")
    keystoreAliasPSW = properties.getProperty("keystore.alias_password")
    keyfile = file(keystoreFilepath)
}

android {
    ...

    defaultConfig {
        ...
    }

    signingConfigs {
        myConfig {
            storeFile keyfile
            storePassword keystorePSW
            keyAlias keystoreAlias
            keyPassword keystoreAliasPSW
        }
    }

    buildTypes {
        release {
            ...
            if (keyfile.exists()) {
                signingConfig signingConfigs.myConfig
            }
        }
    }
...
}
```

这样就达到了隐藏的目的。

具体文件分享在了github上，地址：[signature.config](https://github.com/dxjia/common-tools/tree/master/android-studio/signature.config)

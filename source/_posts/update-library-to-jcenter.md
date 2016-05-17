title: 使用Gradle发布Android开源项目到JCenter
tags:
  - Android
  - Bintry
  - jCenter
categories:
  - Programmer
date: 2016-05-05 09:03:24
---
有时候为了别人使用方便，需要将自己的开源库发布到`jcenter`，这样，别人在`build.gradle`里一句话就可以引用到你的库，本文就来介绍如何通过配置gradle来方便的进行library发布。
<!--more-->
之前发布过一个`BaiduVoiceHelper`，后来时间长了，过程就忘记了，重新检索，根据 http://blog.csdn.net/maosidiaoxian/article/details/43148643 这篇文章的指引最后成功啦，原作者还配了写好的`gradle`并开源，地址：https://github.com/msdx/gradle-publish ，实际使用过程中发现其github上的写好的`gradle`是有更新过的，所以跟他的博文稍微有些出入，需要两者结合起来看，所以这里重新总结一下，以便日后直接使用。
# 配置`bintray`账号与`API Key`
首先注册`bintray`账号，地址：https://bintray.com/ 
支持使用`github`账号直接创建bintray账号，账号生成后会自动为你分配一个`API Key`，`账号名`以及`API Key`是我们能够上传库到bintray的钥匙。
登录bintray网站后，先**点击自己的用户名进入个人页面，然后点击`Edit`进入`profile`页面，这里就可以看到API Key了**。
![click user](http://dxjia.cn/wp-content/uploads/2015/09/bintray-user-1.png)
进入个人页面：
![bintray index](http://dxjia.cn/wp-content/uploads/2015/09/bintray-user-2.png)
进入profile：
![bintray profile](http://dxjia.cn/wp-content/uploads/2015/09/bintray-apikey.png)

至此，便可以复制到`API Key`信息啦。

**`接下来`**，我们需要将这些信息存到本地，也就是你的系统`.gradle`目录，这里要注意，我们保存在系统下，而**不是你的project下的.gradle目录**，如果你的是XP，那么一般是在 **`C:\Documents and Settings\用户名\.gradle`**，而如果是win7以上，那么是在**`c:\Users\用户名\.gradle`**。
然后再这个目录下新建`gradle.properties`文件，在其中记录bintray信息，如下：
```
BINTRAY_USER=dxjia
BINTRAY_KEY=xxxxxxx
```
如果已经存在，那么就直接在文件末尾添加上面的内容，具体修改为你自己的实际情况。

>` 这样处理的好处是：我们只需要记录一次，以后每个需要发布到jcenter的库都可以使用到这里的信息，并且这个由于是存放在project目录之外的，所以也不会受到版本控制的影响，不会因为意外上传而泄露了个人信息。`


# 配置项目Gradle
## 复制gradle.properties到待发布module目录
将 https://github.com/msdx/gradle-publish/blob/master/gradle.properties 这个文件复制到你需要发布的库的目录下，`注意是你的待发布的库的module下`，并修改它的内容，如下：
以下面的最终引用方式为例：
```
dependencies {
    compile 'cn.dxjia:imagetextbutton:1.0.0'
}
```
配置内容如下
```
PROJ_GROUP=cn.dxjia
PROJ_VERSION=1.0.0
PROJ_NAME=imagetextbutton
PROJ_WEBSITEURL=https://github.com/dxjia/ImageTextButton
PROJ_ISSUETRACKERURL=
PROJ_VCSURL=git@github.com:dxjia/ImageTextButton.git
PROJ_DESCRIPTION=android button with icon and text
PROJ_ARTIFACTID=imagetextbutton

DEVELOPER_ID=dxjia
DEVELOPER_NAME=dex.jia
DEVELOPER_EMAIL=jdxwind@dxjia.cn
```
最终的引用形式会是 `PROJ_GROUP:PROJ_ARTIFACTID:PROJ_VERSION`的拼接。每次新版本的发布都要记得来修改 `PROJ_VERSION`属性，否则····
## 修改待发布目录的build.gradle
配置好上面的内容之后，接下来我们需要修改工程待发布库目录下的`build.gradle`文件。
在`build.gradle`文件的`dependencies`中增加下面的内容：
```
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.6'
        classpath "org.jfrog.buildinfo:build-info-extractor-gradle:4.0.0" // Remove it if you won't to publish SNAPSHOT version.
```
然后在文件末尾加上下面一句引用：
```
// this script was used to upload files to bintray.
apply from: 'https://raw.githubusercontent.com/msdx/gradle-publish/master/bintray.gradle'
```
这句话可以在gradle build的时候，直接从github仓库拉取`bintray.gradle`文件进行引用，有时候由于网络问题，会出现下面的问题，
```
Gradle sync failed: Software caused connection abort: recv failed
```
这时候可以手动将`bintray.gradle`保存到项目本地待发布的库目录下，并将同目录下的build.gradle最后一句改为：
```
apply from: 'bintray.gradle'
```
这样就可以了。

# 执行发布
至此，我们就可以执行发布命令啦，使用**`命令行`**切换到项目目录下，然后运行：
```
gradlew.bat bintrayUpload
```

或者在Android studio里，直接右侧选择执行 `bintrayUpload` 这个task即可，这个命令执行结束后，就会直接将生成的 `aar`库推送到了你的`bintray`账号下。

# 包含到Jcenter
执行完步骤三后，登录到你的`bintray`个人页面下，也就是 https://bintray.com/dxjia ，会在右下角显示出你的动态，会有刚刚推送的库的记录，点击打开。
![bintray lastest activity](http://dxjia.cn/wp-content/uploads/2015/09/bintray-lastest-activity.png)
打开的页面中：
点击右下角的 **`Add to Jcenter`**按钮，弹出的页面中填写上一些描述，或者不填，然后**`Send`**，如此便发出了申请，一般很快就会审核通过。
![add to jcenter](http://dxjia.cn/wp-content/uploads/2015/09/add-to-jcenter.png)
填写申请描述:
![compose comments](http://dxjia.cn/wp-content/uploads/2015/09/add-to-jcenter-2.png)



>  **`基本上按照上面的4个步骤即可，感谢作者 msdx 提供的现成的脚本。`**



## Reference
[1] http://blog.csdn.net/maosidiaoxian/article/details/43148643
[2] https://github.com/msdx/gradle-publish


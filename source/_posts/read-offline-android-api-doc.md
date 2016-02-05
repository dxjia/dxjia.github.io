title: 如何快速打开Android SDK离线文档
tags:
  - Android
  - API
  - Doc
categories:
  - Programmer
date: 2016-02-05 15:31:34
---
一般我们更新Android SDK都会附带更新该版本对应的`docs`，也就是离线android api文档，但是，但是，尽管是离线的，你会发现在国内打开还是那么慢，就看浏览器一直在那刷。。。这是因为这个离线文档里还有好多脚本会访问google网站，在国内可想而知。。。

下面就介绍如何`秒开`的方法：
<!--more-->

如果你的浏览器是IE，那么直接开启IE的离线模式，再开始浏览文档即可。
![IE Offline Mode](http://7xqitw.com1.z0.glb.clouddn.com/blog/images/read-offline-api-doc4.png)

如果你的浏览器不是IE，可以找一下是否有离线模式，自行百度吧。

我自己用的是google chrome浏览器，可是这浏览器没有离线模式，可以自行找一些插件来曲线救国实现离线模式，这里介绍一下我使用的方法：

有人专门写了一个插件放在了github上，链接：<https://github.com/xesam/android_offline_doc_plugin> ，使用方法如下：

### 下载插件
  直接`git clone`他的项目，或者`Download ZIP`下到本地;
  ![Download ZIP](http://7xqitw.com1.z0.glb.clouddn.com/blog/images/read-offline-api-doc1.png)
  然后将`plugin`目录复制到一个安全的地方，下一步需要将它加载到浏览器上，所以如果目录被删，会造成插件不可用，我是放到了 chrome的程序目录下了；
  ![plugin](http://7xqitw.com1.z0.glb.clouddn.com/blog/images/read-offline-api-doc2.png)
### google chrome 开发者模式安装插件
  直接参照作者的图，如下
  ![install](http://7xqitw.com1.z0.glb.clouddn.com/blog/images/read-offline-api-doc3.png)
  按照上面的红色圈圈步骤一步步来，第`2`步时选择刚刚的`plugin`目录；

### 点击插件开启屏蔽模式
  按照上一步操作之后，F5 刷新一下页面，就会有 插件图标了，点击一下，出现红字`ON`在图标上时，说明OK了。这时候就可以直接将你的离线文档拖到浏览器里浏览啦。。。

这样就可以秒开离线文档了，但是。。。这种方式有个缺点就是  `搜索功能`是被废掉的。如果先使用搜索，要么科学上网直接访问官网，或者访问国内的一些镜像站，比如 <http://www.android-doc.com/reference/packages.html> ,但这些站点内容无法保证同步更新的，所以可以两者结合起来看，而如果能科学上网，Android API文档就不是个事啦。。


> 还是期待 google 回归啊...... 


**[Reference]**
    【Chrome】扩展——Android离线文档 : <http://my.oschina.net/xesam/blog/283740>
     插件地址: <https://github.com/xesam/android_offline_doc_plugin>



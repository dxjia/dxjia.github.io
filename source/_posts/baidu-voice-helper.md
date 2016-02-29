title: 百度语音识别(Baidu Voice) Android studio版本
tags:
  - Android
  - Java
categories:
  - Programmer
date: 2016-02-29 09:38:03
---
　最近在一个练手小项目里要用到语音识别，搜索了一下，比较容易集成的就算Baidu voice跟讯飞语音了，baidu提供了直接可以使用的显示控件，而讯飞需要自己实现，另外baidu提供每天5W次的调用频率，对于我来说足够使用啦。所以就选择使用Baidu Voice(控件会有baidu logo和关键字，所以正式产品使用要斟酌)。
<!--more-->
　　看了一下baidu提供的android sdk，还是eclipse时代的，如果想要使用他的控件，需要集成他的资源文件到自己的工程目录，还需要在AndroidManifest.xml里增加权限以及activity、service声明等，有些繁琐，而且这些文件夹杂在你的工程里，多少有些凌乱。
　　另外，有一点，baidu提供的这个控件必须要自己手动设置提示音文件，不设置的话，sdk会报null point错。
```
intent.putExtra(EXTRA_SOUND_START, R.raw.bdspeech_recognition_start);
intent.putExtra(EXTRA_SOUND_END, R.raw.bdspeech_speech_end);
intent.putExtra(EXTRA_SOUND_SUCCESS, R.raw.bdspeech_recognition_success);
intent.putExtra(EXTRA_SOUND_ERROR, R.raw.bdspeech_recognition_error);
intent.putExtra(EXTRA_SOUND_CANCEL, R.raw.bdspeech_recognition_cancel);
```
　　这也是因为目前sdk的jar无法自己包含res文件的原因，所以基于此，我就将他的sdk移植到了android studio上，将这些资源文件以及jar包 so文件统统打包到一个`aar文件`，并另外提供了一个接口文件（只有几个接口，用来调用控件），api方式的开发也可以使用这个aar包，因为其内部包含了baidu的jar包，所以baidu的api都是可以引用到的。

　　库已经分享在github上了，并且也上传到了jcenter，可以通过在build.gradle文件里简单的添加dependencies就可以引用到，不过电脑得联网哦；也可以下载aar包，然后离线使用，具体可以参照我在github上的的readme使用。
　　
　　GitHub：[BaiduVoiceHelper]( https://github.com/dxjia/BaiduVoiceHelper)


title: PHP的require路径问题
tags: PHP
categories:
  - Programmer
date: 2016-02-25 17:01:15
---
在使用PHP为手机app编写API接口时发现，有时候require文件路径无法正常工作，而且显得莫名其妙，感觉不可思议。后来发现，PHP的require文件路径问题还真的是有点奇葩的。
<!--more-->

可以看下这两篇文章，解释的很详细啦：
http://cuckoosnest.iteye.com/blog/479401
http://www.cnblogs.com/rainman/p/4177302.html

也就是说PHP并不是像java或者c++一样，路径永远是相对于当前文件的，那么为了习惯，PHP中也是可以做到的，上面的两篇文章中也都提到啦，方法就是利用`__FILE__`这个魔术变量变量构造绝对路径，这样就永远是相对于当前路径来require了。

```
$currentDir = dirname(__FILE__);

require_once($currentDir.'/Toro.php');
require_once($currentDir.'/apikey.php');
require_once($currentDir.'/ErrorCodes.php');
```


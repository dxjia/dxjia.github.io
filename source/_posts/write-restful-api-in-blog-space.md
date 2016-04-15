title: 在博客空间下为手机APP创建RESTFUL API
tags:
  - PHP
  - RESTFUL
  - API
categories:
  - Programmer
date: 2016-04-14 21:30:28
---
这个blog建立有一段时间了，空间是购买的万网（现在归阿里云）的虚拟空间，OS是linux虚拟主机，之前使用wordpress直接搭建的，后来改为hexo静态博客，当时并没有多想，只想着早点搭起博客，所以将wordpress直接安装在空间的 ** 根目录 ** 下了。
而最近自己在练习做一个Android客户端，有完整的后台接口和APP需求文档，后台的接口为APP提供登录、注册、信息获取等一系列操作。再去买空间不太现实，所以只能将接口实现在blog同目录下，以空间子目录的形式进行实现。
<!--more-->
# RESTFUL API
为了是接口看起来美观易用，采用RESTFUL API设计原则，所谓RESTFUL，我的简单理解，就是接口清晰明了，目录名就携带了要请求的信息，而不再使用庞杂的参数来携带信息，而且URL的设计上都使用名词，举个例子：

**实现一个接口，获取后台`id`为`10001`的用户信息**
> 以前的方式： http ://dxjia.cn/api**`?action=get_user&id=10001`**
> RESTFUL   ： http: //dxjia.cn/api/**`user/10001`**

是不是简洁明了很多。当然具体RESTFUL的设计思想还有更多，这里不做具体介绍啦，可以参考文后的参考文献[ 1 ](#示例)。

# 创建子域名及绑定
为了使url更加美观，我选择使用子域名`api.dxjia.cn`来指向，子域名的创建是非常简单的，购买过域名之后，子域名可以免费创建，这里以我在阿里云的后台管理进行创建为例：
- 首先进入你的域名后台点击解析，直接添加解析，如下图，记录值部分写你的主机(虚拟空间)真实IP地址。一般都同时添加不带www和带www。
![Img](http://dxjia.cn/wp-content/uploads/2015/08/ziyuming.png)
- 然后进入主机(虚拟空间)后台进行域名绑定，绑定域名到空间。
![Img](http://dxjia.cn/wp-content/uploads/2015/08/yumingbangding.png)
这样之后，api.dxjia.cn就会指向你的空间根目录，是的，现在还没有完成子域名指向要存放我们实现的API的代码的根目录。需要继续往下看。

# 创建子目录
在网站根目录下创建子目录`api`，为了以后api的版本升级在api目录下继续创建子目录 v1 和 v2，用来区别使用的api版本，也是很时髦的功能。如下：
![Img](http://dxjia.cn/wp-content/uploads/2015/08/mulujiegou.png)

其中`htdocs`是空间根目录，`wp_`开头的目录都是wordpress的目录。
当然，如果你不需要区分版本v1和v2，那么就可以直接在api目录下来写接口代码啦。
# 指向子目录
子域名和子目录都创建好了，而且子域名这时候也已经绑定到空间了，那么我们现在访问api.dxjia.cn，发现其依旧是打开我的blog了，这是因为我们还没有将子域名指向我们的子目录，其默认直接执行根目录下的index.php，也就是blog的入口啦。
至于如何将子域名指向子目录，网上有很多都是通过修改`.htaccess`文件来达到重定向的，确实是可以做到的，但我对正则表达式比较头晕，所以选择了另外一种方式：

** 既然访问`api.dxjia.cn`是会执行空间根目录下的`index.php`文件，那么我们在这个文件里区分访问的域名，如果是api.dxjia.cn这个子域名，我们就直接跳到api子目录下的php文件；如果是dxjia.cn，那么我们才执行wordpress博客代码 **

```php
/**
 * change this to false to disable restful api
 */
$apiEnable = true;

$requireHost = strtolower($_SERVER['HTTP_HOST']);
$isRequiringApi = $apiEnable & (!strcmp($requireHost, "api.dxjia.cn") || !strcmp($requireHost, "www.api.dxjia.cn"));

if ($isRequiringApi) {
    require ("api/api.php");
} else {
/**
 * Front to the WordPress application. This file doesn't do anything, but loads
 * wp-blog-header.php which does and tells WordPress to load the theme.
 *
 * @package WordPress
 */

/**
 * Tells WordPress to load the WordPress theme and output it.
 *
 * @var bool
 */
define('WP_USE_THEMES', true);

/** Loads the WordPress Environment and Template */
require( dirname( __FILE__ ) . '/wp-blog-header.php' );
}
```

else 分支下的代码是原先博客的执行逻辑，这样就一目了然了，如果判断是访问的子域名`api.dxjia.cn`或`www.api.dxjia.cn`那么我们就直接require api目录下的`api.php`代码，这样就解决了跳转的问题。

# 入口文件
再来看看api.php的写法，其中利用与index.php中差不多的方式来进行v1还是v2的api访问控制。
```php
$request_method = strtolower($_SERVER['REQUEST_METHOD']);
$path_info = '/';

if (! empty($_SERVER['PATH_INFO'])) {
    $path_info = $_SERVER['PATH_INFO'];
} elseif (! empty($_SERVER['ORIG_PATH_INFO']) && $_SERVER['ORIG_PATH_INFO'] !== '/api.php') {
    $path_info = $_SERVER['ORIG_PATH_INFO'];
} else {
    if (! empty($_SERVER['REQUEST_URI'])) {
        $path_info = (strpos($_SERVER['REQUEST_URI'], '?') > 0) ? strstr($_SERVER['REQUEST_URI'], '?', true) : $_SERVER['REQUEST_URI'];
    }
}

if(strstr($path_info,"/v1")) {
    require("v1/index.php");
} elseif(strstr($path_info,"/v2")) {
    require("v2/index.php");
}
```

# 使用Toro实现RESTFUL API
通过上面的步骤，我们已经能够使` http: //api.dxjia.cn/v1/`这样的访问正确跳转到`api/v1/index.php`上来了，接下来就是要实现restful接口了。
因为我的接口并不是很多，所以选择了轻量的ToroPHP开源库，其github地址：https://github.com/anandkunal/ToroPHP ，该库代码只有100多行，利用了router与动态访问特性（类似java的反射），可以非常轻松的实现RestFul接口。具体可以查看他的开源介绍。
## 引入ToroPHP
- 将https://github.com/anandkunal/ToroPHP/tree/master/src 下的Toro.php复制到v1目录下，并在该目录下的index.php中进行require。
- 修改Toro.php，去掉访问路径中的/v1，** 如果是在api目录下，不引入v1和v2目录的话，其实直接将Toro.php复制过来就可以正常工作啦 **，但引入v1和v2目录后，要想让Toro在`api/v1`和`api/v2`目录下正常工作，需要去掉访问路径中的`/v1`和`/v2`，如下：

```php
        $path_temp = $path_info;
        $checkHead = substr($path_temp, 0, 3);
        if ($checkHead) {
            if (strstr($checkHead, '/v1')) {
                // 截取掉/v1
                $path_info = substr($path_temp, 3);
            }
        }

        if (strlen($path_info) === 0) {
            $path_info = "/";
        }
```

## 创建handler
ToroPHP的设计原则是一个url对应一个handler，比如对于http: //api.dxjia.cn/v0/这个url我们让其指向HelloWorld.php，那么我们需要创建这个文件，并单独一个类：
```php
<?php
class HelloHandler {
    function get() {
      echo "Hello, world";
    }
}
```

等建立关联后，使用url访问，会直接自动调用这里的get方法。
当然也可以建立post，具体可以参照它的页面。
## 建立路由
创建好handler之后，我们需要在index.php中为其与url建立路由关系，很简单如下：

```php
require("handlers/HelloHandler.php");
require("Toro.php");

Toro::serve(array(
    "/" => "HelloHandler"
));
```

路由关系都是在一起建立的，如下还增加了其他接个API接口：

```php
require("handlers/HelloHandler.php");
require("handlers/UserHandler.php");
require("handlers/UsersHandler.php");
require("handlers/BookHandler.php");

require("Toro.php");

Toro::serve(array(
    "/" => "HelloHandler",
    "/users" => "UsersHandler", // 获取所有用户列表
    "/users/:number" => "UserHandler", // 获取指定id的用户信息
    "/users/:number/book/:alpha" => "BookHandler" // 获取指定id的用户所拥有的指定名字的book信息
));
```

更多的使用方法请参考我的示例，或者ToroPHP的官网
# 示例
因为v1目录本身我自己正在使用，所以这里使用`v0`目录来进行示例，

```
http://api.dxjia.cn/v0
http://api.dxjia.cn/v0/users
http://api.dxjia.cn/v0/users/10001
http://api.dxjia.cn/v0/users/10001/book/God
```

代码共享在github上了：[restful-api-example-use-torophp](https://github.com/dxjia/restful-api-example-use-torophp)


# Reference
[ 1 ] 《RESTful API 设计指南》 http://www.ruanyifeng.com/blog/2014/05/restful_api.html
[ 2 ] 《ToroPHP》https://github.com/anandkunal/ToroPHP



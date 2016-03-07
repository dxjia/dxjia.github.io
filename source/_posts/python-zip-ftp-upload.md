title: 使用python自动打包hexo网站并上传
tags:
  - hexo
  - python
  - blog
  - ftp
categories:
  - Programmer
date: 2016-03-07 11:18:56
---
网页空间是买的虚拟主机，不支持git部署，每次写完博客，都需要自己手动上传到空间，尽管hexo有ftpsync的插件可以做ftp上传，但用了几次总是有问题，而且我的网站也有在github上部署，配置在_config.xml里会泄漏ftp信息。自己手动上传也有诸多问题，首先遍历比较多，慢不说，还比较容易出现文件错误，尤其中文文件。所以后来就先在本地手动打包为一整个zip压缩包，再上传到空间，然后再去空间解压覆盖，这样比较安全放心。
下面介绍我目前使用的这种方式，利用phthon脚本打包hexo public(生成的整个网页文件)目录，然后上传。
<!--more-->
## 新建 zip-ftp-deploy.py
代码如下：
```
#encoding: utf-8
__author__ = 'dxjia'
__mail__ = 'jdxwind@dxjia.cn'
__date__ = '2016-03-07'
__version = 1.0

import os, os.path
import zipfile
from ftplib import FTP

#全局变量
PUBLIC_FOLDER_NAME = 'public'
#目标压缩包文件名
TARGET_ZIP_FILE_NAME = 'a-ftp-deplog.zip'
#FTP参数
FTP_IP = "网站FTP IP地址"
FTP_USER_NAME = '用户名'
FTP_PASSWORD = '密码'
FTP_TARGET_FOLDER = 'htdocs' #网站目录

#打包函数
def zip_dir(dirname, zipfilename):
    filelist = []
    if os.path.isfile(dirname):
        filelist.append(dirname)
    else :
        for root, dirs, files in os.walk(dirname):
            for name in files:
                filelist.append(os.path.join(root, name))
         
    zf = zipfile.ZipFile(zipfilename, "w", zipfile.zlib.DEFLATED)
    for tar in filelist:
        arcname = tar[len(dirname):]
        zf.write(tar,arcname)
    zf.close()
 
if __name__ == '__main__':
    public_path = os.path.join(os.getcwd(), PUBLIC_FOLDER_NAME)
    zip_file_path = os.path.join(os.getcwd(), TARGET_ZIP_FILE_NAME)
    if os.path.exists(public_path):
        zip_dir(public_path, zip_file_path)
    else:
        print "have no public folder, please excute \'hexo g\' first"

    if os.path.exists(zip_file_path):
        ftp = FTP(FTP_IP)
        ftp.login(FTP_USER_NAME, FTP_PASSWORD)
        ftp.cwd(FTP_TARGET_FOLDER)
        f = open(zip_file_path, 'rb')
        ftp.storbinary('STOR %s' % TARGET_ZIP_FILE_NAME, f)
        f.close()
        ftp.close()
        os.remove(zip_file_path)
    else:
        print "failed"
```
在全局变量部分配置好你的FTP信息，然后使用python执行这个脚本就可以了。
```
python zip-ftp-deploy.py
```
## 使用
这样，在每次写完新的文章之后，按照下面的步骤来操作：
- hexo g -d     // 生成网页，并部署到github
- git add ./
- git commit -m "new post"
- git push      // 上传hexo源码到github
- python zip-ftp-deploy.py     // 打包public目录，并上传到网站空间

然后每隔一段时间，登陆到网站后台，解压缩你上传的zip包到网页根目录即可更新。
> 注意：别忘记将`zip-ftp-deploy.py`加入 gitignore哦。。。

## 源码
上传在GitHub上：[python-ftp-deploy](https://github.com/dxjia/common-tools/blob/master/python-ftp-deploy/zip-ftp-deploy.py)
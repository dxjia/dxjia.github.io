title: git and repo 
tags:
  - Android
  - Java
categories:
  - Programmer
date: 2016-11-01 18:52:33
---
git 和 repo 的关系？
<!-- more -->
# git
git 是一个免费开源的版本控制系统，可以非常方便的对项目进行协同开发，分支和版本管理。
在做 Android 源码开发的时候，可以看到，Android 的源码库分成了很多的 git 库，基本上每个模块就会单独一个 git 库，这样的好处也是非常明显的，首先是非常清晰，Android 系统体系十分庞大，这样的划分清晰明了，每个 git 库都互不干扰，另一方面，这样也方便了以 Android 源码来进行二次开发的版本管理与修改追溯。

# repo 
每个git库内的版本控制都可以由 git 完成，那么谁来管理所有的 git 库呢，那就是 repo 了，repo 是 google 编写的脚本，是用 python 编写的，来统一管理所有 git 仓库，当然其中肯定是调用了 git 了的。



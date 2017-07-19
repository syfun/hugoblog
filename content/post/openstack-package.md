+++
date = "2015-11-16T18:55:12+08:00"
description = ""
title = "Openstack Deb打包"
thumbnail = ""
tags = ["云计算"]
categories = ["云计算"]
draft = true
+++



前两年已经学过如何打包，当时打包打的很6啊，但是时间一长就忘了。为了再次忘记的时候有个东西翻阅，所以写下这篇博客。

首先给出几篇参考文章：

1. [一分钟学会将OpenStack Havana代码编译成DEB包](http://blog.csdn.net/xiaoquqi/article/details/17681003)
2. [Debian 新维护人员手册](http://www.debian.org/doc/manuals/maint-guide/index.zh-cn.html)

<!--more-->

Note：一些命令的安装包。

```
apt-get install -y debootstrap equivs schroot devscripts build-essential checkinstall sbuild \
   dh-make  bzr bzr-builddeb  git python-setuptools  
```


## 获取源码

打包之前你需要有源码，不论你是从官方的Github或者自己的仓库克隆的，还是直接Launchpad下载的tar包，又或者从其他地方搞来的，你都得有哈。我们这里以官方的Github为例。

我比较熟悉cinder，正好最近也出了L版了，就用cinder的L版来示范如何打包了。

```bash
mkdir tmp && cd tmp
git clone https://github.com/openstack/cinder.git
cd cinder && git checkout -b liberty origin/stable/liberty
```

切换到L版后就需要先生成源码包。

```bash
python setup.py sdist
```

源码包在dist目录下，名字叫。源码包做好之后先不管，我们继续下面的步骤。

## 获取debain

deb打包需要一个叫debian的文件夹，里面是一些控制文件，我们通过这些文件来定制软件包行为。我们不需要自己去创建这些文件，Launchpad已经为我们提供了。使用bzr命令进行下载。

```bash
bzr branch lp:~ubuntu-server-dev/cinder/liberty
```

这样我们得到了liberty目录，其下一层就是debain。然后我们自己打的包肯定要自定义自己的版本号。

```bash
cd liberty
dch -b -D trusty --newversion "1:2015.11.16-0ubuntu1" "Test."  
debcommit
```

打开debain/changelog，我们能够看到新增加了一条记录。

```
cinder (1:2015.11.16-0ubuntu1) trusty; urgency=medium

  * Test.

 -- Firstname Lastname <your.email.address@example.org>  Mon, 16 Nov 2015 11:42:06 +0800
```

`-D trusty`制定了安装包的平台是ubuntu 14.04。`newversion`就是版本号了。

除了使用dch命令之外，也可以直接修改changelog，只要按照上面的形式添加记录就可以。对记录内容不清楚的可以看看上面的第二篇参考文章中得[changelog](http://www.debian.org/doc/manuals/maint-guide/dreq.zh-cn.html#changelog)。在这里说明下如果是手动改，需要在`debcommit`之前`bzr add .`一下，是不是看着很眼熟，和git是一样的，这说明bzr也是一款代码仓库管理工具。

debain目录也暂时到这，下面就是激动人心的打包了！


## 打包

首先我们将dist目录下得源码包名称改为cinder_2015.11.16.orig.tar.gz，这是为了和changelog中得版本对应，需要添加orig，需要添加orig，需要添加orig，重要的事情说三遍。然后将源码包拷到liberty目录的上一层目录中，使其和liberty同级。然后我们需要安装打包的一些依赖包。

```
cd liberty
mk-build-deps -i -t 'apt-get -y' debian/control

# 这里注意一下，因为打包的过程中会运行单元测试，所以需要安装一些依赖包
cd cinder
pip install -r requirements.text
pip install -r test-requirements.txt
```

Note：`mk-build-deps`会生成一个包含`debian/control`中所有依赖包的整合包，直接拷到其他地方。不然如果`bzr add .`了之后会把这个包加到bzr的仓库中，这样就会遇到其他问题，所以还是直接拷出来的好。

安装完之后就可以正式打包了。
```
cd liberty
bzr builddeb -- -sa -us -uc  
```

然后。。。。。。然后我们就遇到了问题：

```
Building using working tree
Building package in merge mode
Looking for a way to retrieve the upstream tarball
Using the upstream tarball that is present in /root/workspace
bzr: ERROR: An error (1) occurred running quilt: None

Applying patch skip-rtslib-test.patch
can't find file to patch at input line 3
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|--- a/cinder/tests/test_cmd.py
|+++ b/cinder/tests/test_cmd.py
--------------------------
No file to patch.  Skipping patch.
2 out of 2 hunks ignored
Patch skip-rtslib-test.patch does not apply (enforce with -f)
```

看了下发现是补丁的问题，去pacthes目录看了下，这些补丁都过时了，所以我们可以去掉这些补丁。方法很简单，将patches目录下的series文件清空。

继续打包的时候在单元测试的地方遇到了问题。


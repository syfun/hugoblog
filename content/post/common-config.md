+++
tags = ["其他"]
categories = ["其他"]
date = "2017-03-08T19:08:50+08:00"
description = ""
title = "常用配置"
thumbnail = ""
draft = true
+++

记录常用配置。

<!--more-->


#### Pip源配置

```shell
# linux/Mac: ~/.pip/pip.conf
# Windows: %HOME%\pip\pip.ini
[global]
index-url = https://pypi.douban.com/simple
```

##### NPM源配置

```shell
# 1. config 指定
npm config set registry https://registry.npm.taobao.org 
npm info underscore （如果上面配置正确这个命令会有字符串response）

# 2. 命令行指定(暂时)
npm --registry https://registry.npm.taobao.org install bower

# 3. ~/.npmrc
registry = https://registry.npm.taobao.org
```




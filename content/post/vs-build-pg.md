+++
categories = ["其他"]
date = "2015-03-10T19:24:26+08:00"
description = ""
title = "使用VS2013编译安装postgresql"
thumbnail = ""
tags = []

+++


整理一下在windows平台下使用vs2013编译安装postgresql的步骤，postgresql版本是9.4.0。

我的pg源码目录是E:\Workspace\postgresql-9.4.0。

<!--more-->

首先修改src\tools\msvc下面的Mkvcbuild.pm文件：

```c
#my $vsVersion = DetermineVisualStudioVersion();
my $vsVersion = '12.00';

$solution = CreateSolution($vsVersion, $config);
```

安装perl、tcl、git, [windows版本的下载链接](http://pan.baidu.com/s/1dDhGMJ3)。
安装完成之后删除掉C:\Program Files (x86)\Git\bin下的perl和tcl(删除这一步一定要做，同时，
如果你的机器安装了MinGW或者Cygwin，请先将其从环境变量PATH中去除)。

打开vs2013命令工具提示，进入源码目录，运行

```bash
build DEBUG
```

编译过程会花很长时间，请耐心等待。完成后，就可以进行安装了。然后同目录下运行install命令：

```bash
install E:\Workspace\postgresql-9.4.0\dbtest
```

进入到dbtest目录，进行数据库的初始化:

```bash
bin\initdb.exe -D data -U postgres -W
```

打开根目录下的psql.sln工程, 修改src\port\pg\_config\_paths.h

```c
#define PGBINDIR "/bin"
#define PGSHAREDIR "E:\\Workspace\\postgresql-9.4.0\\dbtest\\share"
#define SYSCONFDIR "/etc"
#define INCLUDEDIR "/include"
#define PKGINCLUDEDIR "/include"
#define INCLUDEDIRSERVER "/include/server"
#define LIBDIR "E:\\Workspace\\postgresql-9.4.0\\dbtest\\lib"
#define PKGLIBDIR "E:\\Workspace\\postgresql-9.4.0\\dbtest\\lib"
#define LOCALEDIR "/share/locale"
#define DOCDIR "/doc"
#define HTMLDIR "/doc"
#define MANDIR "/man"
```

修改postgres项目的属性，添加命令参数：

![postgres属性](postgres.png)

然后就可以使用postgres项目进行调试跟踪代码了。

+++
categories = ["云计算"]
date = "2014-08-09T19:16:16+08:00"
description = ""
title = "Paste deployment(转)"
thumbnail = ""
tags = ["Python"]

+++



转载自[Python.Paste指南之Deploy](http://blog.csdn.net/ztejiagn/article/details/8722765)

Paste.Deploy主要是用来载入WSGI中的Web App应用，其核心函数是loadapp(),下文中PD就指代Paste.Deploy。

<!--more-->

## 简介及安装

Paste Deployment是一种机制，通过loadapp函数和一个配置文件或者egg包来载入WSGI应用。可以通过Pip或者easy_install安装。

## 配置文件Config Flie

一个配置文件后缀为ini，内容被分为很多段（section），PD只关心带有前缀的段，比如[app:main]或者[filter:errors]，总的来说，一个section的标识就是[type:name],不是这种类型的section将会被忽略。<br/>
一个section的内容是以键=值来标示的。#是一个注释。在段的定义中，有以下几类：<br/>
\[app:main]:定义WSGI应用，main表示只有一个应用，有多个应用的话main改为应用名字<br/>
\[server:main]:定义WSGI的一个server。<br/>
\[composite:xxx]：表示需要将一个请求调度定向（dispatched）到多个,或者多种应用上。以下是一个简单的例子，例子中，使用了composite，通过urlmap来实现载入多应用。<br/>
\[fliter:]：定义“过滤器”，将应用进行进一步的封装。<br/>
\[DEFAULT]：定义一些默认变量的值。

以下是一个例子：

```ini
[composite:main]
use = egg:Paste#urlmap
/ = home
/blog = blog
/wiki = wiki
/cms = config:cms.ini

[app:home]
use = egg:Paste#static
document_root = %(here)s/htdocs

[filter-app:blog]
use = egg:Authentication#auth
next = blogapp
roles = admin
htpasswd = /home/me/users.htpasswd

[app:blogapp]
use = egg:BlogApp
database = sqlite:/home/me/blog.db

[app:wiki]
use = call:mywiki.main:application
database = sqlite:/home/me/wiki.db
```

下面会进行分段的讲解

###  composite

```ini
[composite:main]
use = egg:Paste#urlmap
/ = home
/blog = blog
/wiki = wiki
/cms = config:cms.ini
```

这是一个composite段，表示这将会根据一些条件将web请求调度到不同的应用。use = egg:Paste#urlmap表示我们奖使用Pasteegg包中urlmap来实现composite，这一个段(urlmap)可以算是一个通用的composite程序了。根据web请求的path的前缀进行一个到应用的映射(map)。这些被映射的程序就包括blog,home,wiki,config:cms.ini（映射到了另外一个配置文件，PD再根据这个文件进行载入）

###  App type1

```ini
[app:home]
use = egg:Paste#static
document_root = %(here)s/htdocs
```

app是一个callable object，接受的参数(environ,start_response)，这是paste系统交给application的，符合WSGI规范的参数. app需要完成的任务是响应envrion中的请求，准备好响应头和消息体，然后交给start_response处理，并返回响应消息体。egg:Paste#static也是Paste包中的一个简单程序，它只处理静态文件。它需要一个配置文件document_root,后面的值可以是一个变量,形式为%（var）s相应的值应该在[DEFAULT]字段指明以便Paste读取。比如：

```ini
[app:test]
use = egg:Paste#static
document_root = %(path)s/htdocs
[DEFAULT]
path = /etc/test
```

### fliter

filter是一个callable object，其唯一参数是(app)，这是WSGI的application对象，filter需要完成的工作是将application包装成另一个application（“过滤”），并返回这个包装后的application。

```ini
[filter-app:blog]
use = egg:Authentication#auth
next = blogapp
roles = admin
htpasswd = /home/me/users.htpasswd

[app:blogapp]
use = egg:BlogApp
database = sqlite:/home/me/blog.db
```

[filter-app:blog]fliter-app字段表明你希望对某个应用进行包装，需要包装的应用通过next指明（表明在下一个段中），这个字段的意思就是，在正式调用blogapp之前，我会调用egg:Authentication#auth进行一个用户的验证，随后才会调用blogapp进行处理。后面的[app:blogapp]则是定义了blogapp，并指明了需要的database参数。

### App type2

```ini
[app:wiki]
use = call:mywiki.main:application
database = sqlite:/home/me/wiki.db
```

这个段和之前的app段定义类似，不同的是对于wiki这个应用，我们没有使用egg包，而是直接对mywiki.main这个模块中的application对象使用了call方法。python，中一切皆对象，作为WSGI app的可以是一个函数，一个类，或者一个实例，使用call的话，相应的函数，类，实例中必须实现call()方法。此类app的格式用冒号分割: call(表示使用call方法):模块的完成路径名字:应用变量的完整名字

##  基本使用

PD的主要使用就是通过读取配置文件载入WSGI应用。如下:

```python
from paste.deploy import loadapp
wsgi_app = loadapp('config:/path/to/config.ini')
```

注意，这里需要指明绝对路径。

## 更多关于配置文件

### App

单个配置文件中可以定义多个应用个，每个应用有自己独立的段。应用的定义以[app:name]的格式，[app:main]表示只有一个应用。应用的定义支持以下五种格式：

```ini
[app:myapp]
use = config:another_config_file.ini#app_name
#使用另外一个配置文件

[app:myotherapp]
use = egg:MyApp
#使用egg包中的内容

[app:mythirdapp]
use = call:my.project:myapplication
#使用模块中的callable对象

[app:mylastapp]
use = myotherapp
#使用另外一个section

[app:myfacapp]
paste.app_factory = myapp.modulename:app_factory
#使用工厂函数
```

其中，最后一种方式，将一个app指向了某些python代码。此模式下，必须执行app协议，以app_factory表示，后面的值需要import的东西，在这个例子中myapp.modulename被载入，并从其中取得了app_factory的实例。<br/>
app_factory是一个callable object，其接受的参数是一些关于application的配置信息：(global_conf,\*\*kwargs)，global_conf是在ini文件中default section中定义的一系列key-value对，而**kwargs，即一些本地配置，是在ini文件中，app:xxx section中定义的一系列key-value对。app_factory返回值是一个application对象<br/>
在app的配置中，use参数以后配置就算结束了。其余的键值参数将会作为参数，传递到factory中，如下：

```ini
[app:blog]
use = egg:MyBlog
database = mysql://localhost/blogdb #这是参数
blogname = This Is My Blog! #这是参数
```

### 全局配置

全局配置主要是用于多个应用共用一些变量，这些变量我们规定放在段[DEFAULT]中，如果需要覆盖，可以在自己的app中重新定义，如下：

```ini
[DEFAULT]
admin_email = webmaster@example.com
[app:main]
use = ...
set admin_email = bob@example.com
```

### composite app

composite是一个运行着像是app，但是实际上是由多个应用组成的。urlmap就是composite app的一个例子，url不同的path对应了不同的应用。如下：

```ini
[composite:main]
use = egg:Paste#urlmap
/ = mainapp
/files = staticapp

[app:mainapp]
use = egg:MyApp

[app:staticapp]
use = egg:Paste#static
document_root = /path/to/docroot
```

在loadapp函数的执行中，composite app被实例化，它同时还会访问配置文件中定义的其他应用。

### app定义高级用法

在app段中，你可以定义fliters和servers，通过fliter:和server: PD通过loadserver和loadfilter函数进行调用，工作机制都一样，返回不同的对象。

### filter composition

应用filter的方式很多，重要的是看你filter的数量和组织形式。下面会一一介绍应用fliter的几种方式：

#### 使用filter-with

```ini
[app:main]
use = egg:MyEgg
filter-with = printdebug

[filter:printdebug]
use = egg:Paste#printdebug
# and you could have another filter-with here, and so on...
```

#### 使用fliter-app

```ini
[fliter-app:printdebug]
use = egg:Paste
next = main

[app:main]
use = egg:MyEgg
```

#### 使用pipeline

当使用多个filter的时候需要使用pipeline的方式，它需要提供一个key参数pipeline,后面的值是一个列表，最后以应用结尾。如下：

```ini
[pipeline:main]
pipeline = filter1 egg:FilterEgg#filter2 filter3 app

[filter:filter1]
...
```

假设在ini文件中, 某条pipeline的顺序是filter1, filter2, filter3，app, 那么，最终运行的app\_real是这样组织的：

```python
app_real = filter1(filter2(filter3(app)))
```

在app真正被调用的过程中，filter1.\_\_call\_\_(environ,start\_response)被首先调用，若某种检查未通过，filter1做出反应；否则交给filter2.\_\_call\_\_(environ,start\_response)进一步处理，若某种检查未通过，filter2做出反应，中断链条，否则交给filter3.\_\_call\_\_(environ,start\_response)处理，若filter3的某种检查都通过了，最后交给app.\_\_call\_\_(environ,start\_response)进行处理。

#### 读取配置文件

如果希望在不创建应用的情况下得到配置文件，可以使用appconfig(uri)函数，将会以字典形式返回使用的配置。这个字典包括了全局很本地的配置信息，所以可以通过属性方法获得相应的attributes （.local_conf and .global_conf）

## 其他

###  如何引用Egg包

egg是python的一个包，pip easy_install等都是安装egg包的方式。关注egg包要注意： +某一egg包是有标准说明的<br/>
python setup.py name +有entry point，不用太在意，这个只是说明调用程序的参数。

### 定义factory函数

工厂函数的定义还是遵循之前提到的应用的协议。目前，用于工厂函数的协议有以下：<br/>

- paste.app_factory
- paste.composite_factory
- paste.filter_factory
- paste.server_factory

所有的这些都希望有一个含有\__call\__方法的（函数，方法，类）。

#### paste.app_factory

```python
def app_factory(global_config, **local_conf):
    return wsgi_app
```

global_config是一个字典，而local_conf则是关键字参数。返回一个wsgi_app（含有call方法。）

#### paste.composite_factory

```python
def composite_factory(loader, global_config, **local_conf):
   return wsgi_app
```

loader是一个对象，有几个有趣的方法,get_app(name_or_uri, global_conf=None)根据name返回一个wsgi应用，get_filter（）和get_server（）也是一样。看一个更加复杂的例子，举例一个pipeline应用：

```python
def pipeline_factory(loader, global_config, pipeline):
    # space-separated list of filter and app names:
    pipeline = pipeline.split()
    filters = [loader.get_filter(n) for n in pipeline[:-1]]
    app = loader.get_app(pipeline[-1])
    filters.reverse() # apply in reverse order!
    for filter in filters:
      app = filter(app)
    return app
```

相应的配置文件如下：

```ini
[composite:main]
use = <pipeline_factory_uri>
pipeline = egg:Paste#printdebug session myapp

[filter:session]
use = egg:Paste#session
store = memory

[app:myapp]
use = egg:MyApp
```

#### paste.filter_factory

fliter的工厂函数和app的共产函数类似，除了它返回的是一个filter,fliter是一个仅仅把一个wsgi应用作为唯一参数的callable对象，返回一个被filter了的应用。 以下是一个例子，这个filter会检查CGI中REMOTE_USER变量是否存在，并创建一个简单的认证过滤器。

```python
def auth_filter_factory(global_conf, req_usernames):
    # space-separated list of usernames:
    req_usernames = req_usernames.split()
    def filter(app):
        return AuthFilter(app, req_usernames)
    return filter

class AuthFilter(object):
    def __init__(self, app, req_usernames):
        self.app = app
        self.req_usernames = req_usernames

def __call__(self, environ, start_response):
    if environ.get('REMOTE_USER') in self.req_usernames:
            return self.app(environ, start_response)
    start_response(
            '403 Forbidden', [('Content-type', 'text/html')])
    return ['You are forbidden to view this resource']
```

#### paste.filter_app_factory

 和paste.filter_factory类似，接受一个wsgi应用参数，返回一个WSGI应用，所以如果改变以上代码的：

```python
class AuthFilter(object):
    def __init__(self, app, global_conf, req_usernames):
        ....
```
那么，类 AuthFilter就会作为一个filter_app_factory函数使用。

#### paste.server_factory
与以上不同的是，函数返回的是一个server,一个server也是一个callable对象，以一个WSGI应用作为参数，而后为这个应用服务。

```python
def server_factory(global_conf, host, port):
    port = int(port)
    def serve(app):
        s = Server(app, host=host, port=port)
        s.serve_forever()
    return serve
```

Server的实现用户可以自定义，可以参考python包wsgiref

#### paste.server_runner

与 paste.server_factory类似，不同的是参数格式。

## 代码实践

### 配置文件

```ini
[DEFAULT]
company = UESTC
school = Commuication and Information

[composite:common]
use = egg:Paste#urlmap
/ = showversion
/detail = showdetail

[pipeline:showdetail]
pipeline = filter1 filter2 showstudetail

[filter:filter1]
#filter1 deal with auth,read args below
paste.filter_factory = python_paste:AuthFilter.factory
user = admin
passwd = admin

[filter:filter2]
#filter2 deal with time,read args below
paste.filter_factory = python_paste:LogFilter.factory
#all value is string
date = 20121120

[app:showstudetail]
name = wangchang
age = 23
paste.app_factory = python_paste:ShowStuDetail.factory

[app:showversion]
version = 1.0.0
```

从配置文件可以看出，这个程序会有如下操作：

对于http://localhost/的访问，会调用showversion这个应用，应用读取ini文件中的version值并返回。注意，在ini中的所有值都是字符串。

对于http://localhost/detail的访问，会先经过filter1以及filter2，这两个filter分别处理认证和LOG信息，他们会读取ini配置中的用户信息以及时间。最后才是交给showstudetail处理，showstudetail会读取ini中的用户信息并返回。注意，使用多个filter的时候需要使用pipeline方式。

### 代码

```python
# -*- coding:utf-8 -*-
__author__ = 'sunyu'

import os
from webob import Request
from webob import Response
from paste.deploy import loadapp
from wsgiref.simple_server import make_server


class AuthFilter(object):
    """filter1,auth
       1.factory read args and print,return self instance
       2.call method return app
       3.AuthFilter(app)
    """

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        print 'this is Auth call filter1'
        #pass environ and start_response to app
        return self.app(environ, start_response)

    @classmethod
    def factory(cls, global_conf, **kwargs):
        """global_conf and kwargs are dict"""
        print '######filter1##########'
        print 'global_conf type:', type(global_conf)
        print '[DEFAULT]', global_conf
        print 'kwargs type:', type(kwargs)
        print 'Auth Info', kwargs

        return AuthFilter


class LogFilter(object):
    """filter2,Log"""

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        print 'This is call LogFilter filter2'
        return self.app(environ,start_response)

    @classmethod
    def factory(cls, global_conf, **kwargs):
        #print type(global_conf)
        #print type(kwargs)
        print '######filter2###########'
        print '[DEFAULT]', global_conf
        print 'Log Info', kwargs

        return LogFilter


class ShowStuDetail(object):
    """app"""
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __call__(self, environ, start_response):
        print 'this is call ShowStuDetail'
        start_response("200 OK", [("Content-type", "text/plain")])
        content = []
        content.append("name: %s age:%s\n" % (self.name, self.age))
        content.append("**********WSGI INFO***********\n")

        for k, v in environ.iteritems():
            content.append('%s:%s \n' % (k, v))

        return ['\n'.join(content)]

    @classmethod
    def factory(cls, global_conf, **kwargs):
        #self.name = kwargs['name']
        #self.age = kwargs['age']
        return ShowStuDetail(kwargs['name'],kwargs['age'])


class ShowVersion(object):
    """app"""
    def __init__(self, version):
        self.version = version

    def __call__(self, environ, start_response):
        print 'this is call ShowVersion'
        req = Request(environ)
        res = Response()
        res.status = '200 OK'
        res.content_type = "text/plain"
        content = []

        content.append("%s\n" % self.version)
        content.append("*********WSGI INFO*********")
        for k, v in environ.iteritems():
            content.append('%s:%s\n' % (k, v))
        res.body = '\n'.join(content)
        return res(environ,start_response)

    @classmethod
    def factory(cls, global_conf, **kwargs):
        #self.version = kwargs['version']
        return ShowVersion(kwargs['version'])

if __name__ == '__main__':
    config = "python_paste.ini"
    app_name = "common"
    app = loadapp("config:%s" % os.path.abspath(config), app_name)
    server = make_server('localhost', 7070, app)
    server.serve_forever()
```

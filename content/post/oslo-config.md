+++
thumbnail = ""
tags = ["Python"]
categories = ["云计算"]
date = "2014-07-11T19:15:05+08:00"
description = ""
title = "Openstack oslo.config介绍(转)"

+++


**本文转自[OpenStack源码探秘（二）——Oslo.config](http://blog.csdn.net/networm3/article/details/8946556)**

oslo.config, OpenStack中负责CLI和CONF配置项解析的组件。E版本前，这个功能是放在cfg模块中的，
后来社区中考虑将OpenStack中共性的组件都剥离出来，统一放在oslo模块中。今后开发新的OpenStack组件，估计都要用到Oslo模块。

<!--more-->

下面说明一下用法：
在oslo的cfg模块载入的时候(from oslo.config import cfg)，会自动运行模块中的载入代码CONF = ConfigOpts()，
创建一个全局的配置项管理类。和许多Conf配置模块一样，oslo.conf在使用时，需要先声明配置项的名称、定义类型、帮助文字、缺省值等，
然后再按照事先声明的配置项，对CLI或conf中的内容进行解析。<br />
配置项声明结构示例如下：

```python
common_opts = [
    cfg.StrOpt('bind_host',
           default='0.0.0.0',
               help='IP address to listen on'),
    cfg.IntOpt('bind_port',
               default=9292,
               help='Port number to listen on')
]
```

类型的定义对应Opt的各个子类。<br />
oslo使用register_opt方法，将配置项定义向配置项管理类configOpts的注册是在程序的运行时刻，但是必须在配置项的引用前完成。

```python
CONF = cfg.CONF
CONF.register_opts(common_opts)

port = CONF.bind_port
```

使用conf.register_cli_opts()方法，配置项还可以在管理类ConfigOpts中可选注册为CLI配置项，
通过程序运行的CLI参数中获得配置项取值，并在错误打印时，自动输出给CLI配置项参数的帮助文档。<br />
conf配置文件采用的是ini风格的格式

```python
  glance-api.conf:
    [DEFAULT]
    bind_port = 9292
      ...

    [rabbit]
    host = localhost
    port = 5672
    use_ssl = False
    userid = guest
    password = guest
    virtual_host = /
```

最后通过ConfigOpts类的__call()__方法，执行配置项的解析以及从CLI或配置文件读取配置项的值。

```python
def __call__(self,
             args=None,
             project=None,
             prog=None,
             version=None,
             usage=None,
             default_config_files=None):
```

下面是一个完整的示例

```python
from oslo.config import cfg

opts = [
    cfg.StrOpt('bind_host', default='0.0.0.0'),
    cfg.IntOpt('bind_port', default=9292),
]

CONF = cfg.CONF
CONF.register_opts(opts)
CONF(default_config_files='glance.conf')
def start(server, app):
    server.start(app, CONF.bind_port, CONF.bind_host)
```

OpenStack项目的配置项声明和许多其他开源Python项目一样，配置项声明是放在各个调用的模块里面的。
也就是说哪里用到才到哪里声明。我觉得这种方式是完全体现了Pthonic的一种声明方式，有别于其他方式，
程序员在阅读程序的时候可以非常方便的在文件开头就能找到配置项的声明定义，而不用到某个指定的文件去查找，实现了KISS的原则。

+++
categories = ["Python"]
date = "2015-04-02T19:21:04+08:00"
description = ""
title = "Python argparse"
thumbnail = ""
tags = []

+++


最近在看openstack cinderclient代码，就顺带着看了下argparse。argparse是Python官方推荐的命令行解析工具库。

学习Python库首先要去看[官方文档](https://docs.python.org/2.7/library/argparse.html)，这里有个简单的[Tutorial](https://docs.python.org/2.7/howto/argparse.html#id1)。

<!--more-->

让我们看下Linux中常见的ls命令：

    $ ls
    cpython  devguide  prog.py  pypy  rm-unused-function.patch
    $ ls pypy
    ctypes_configure  demo  dotviewer  include  lib_pypy  lib-python ...
    $ ls -l
    total 20
    drwxr-xr-x 19 wena wena 4096 Feb 18 18:51 cpython
    drwxr-xr-x  4 wena wena 4096 Feb  8 12:04 devguide
    -rwxr-xr-x  1 wena wena  535 Feb 19 00:05 prog.py
    drwxr-xr-x 14 wena wena 4096 Feb  7 00:59 pypy
    -rw-r--r--  1 wena wena  741 Feb 18 01:01 rm-unused-function.patch
    $ ls --help
    Usage: ls [OPTION]... [FILE]...
    List information about the FILEs (the current directory by default).
    Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.
    ...

从上面的四条命令我们可以学到一些概念：

1. ls命令在没有参数时，默认显示当前目录下所有的内容
1. 加了pypy这个位置参数（positional argument），就会得到了pypy目录下的内容
1. 加了-l这个可选参数（optional argument）后，显示文件更多信息
1. 帮助文档

下面我们来看argparse如何使用。

```python
prog.py

import argparse
parser = argparse.ArgumentParser()
parser.add_argument("echo")
args = parser.parse_args()
print args.echo


$ python prog.py
usage: prog.py [-h] echo
prog.py: error: the following arguments are required: echo
$ python prog.py --help
usage: prog.py [-h] echo

positional arguments:
  echo

optional arguments:
  -h, --help  show this help message and exit
$ python prog.py foo
foo
```

- add_argument()方法为程序添加命令行参数，而在加了参数之后，运行程序就需要额外的参数
- parse_args()方法返回接受到的参数，存放在args中，而且它有一个'echo'属性，值便是我们传进去的参数值，是不是很神奇？

上面的程序中，echo参数没有帮助说明，因此我们并不知道它是用来做什么的。

```python
prog.py

import argparse
parser = argparse.ArgumentParser()
parser.add_argument("echo", help="echo the string you use here")
args = parser.parse_args()
print args.echo


$ python prog.py -h
usage: prog.py [-h] echo

positional arguments:
  echo        echo the string you use here

optional arguments:
  -h, --help  show this help message and exit
```

来看一个更有意义的例子：

```python
prog.py

import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", help="display a square of a given number")
args = parser.parse_args()
print args.square**2


$ python prog.py 4
Traceback (most recent call last):
  File "prog.py", line 5, in <module>
    print args.square**2
TypeError: unsupported operand type(s) for ** or pow(): 'str' and 'int'


pyog.py
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", help="display a square of a given number",
                    type=int)
args = parser.parse_args()
print args.square**2


$ python prog.py 4
16
$ python prog.py four
usage: prog.py [-h] square
prog.py: error: argument square: invalid int value: 'four'
```

以上加的参数都是位置参数，下面我们试下可选参数。

```python
prog.py

import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--verbosity", help="increase output verbosity")
args = parser.parse_args()
if args.verbosity:
    print "verbosity turned on"


$ python prog.py --verbosity 1
verbosity turned on
$ python prog.py
$ python prog.py --help
usage: prog.py [-h] [--verbosity VERBOSITY]

optional arguments:
  -h, --help            show this help message and exit
  --verbosity VERBOSITY
                        increase output verbosity
$ python prog.py --verbosity
usage: prog.py [-h] [--verbosity VERBOSITY]
prog.py: error: argument --verbosity: expected one argument
```

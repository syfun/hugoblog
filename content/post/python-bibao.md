+++
description = ""
title = "Python闭包的作用域理解"
thumbnail = ""
tags = ["Python"]
categories = ["Python"]
date = "2015-07-22T22:23:31+08:00"

+++

## 什么是闭包

在维基中，闭包的解释是这样的：


>在计算机科学中，闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。


<!--more-->

我们来看一个简单的实例：

```python
def outer(arg):
    def inner():
        print arg * arg
    return inner

>>> f = outer(2)
>>> f()
4
```

内部函数inner调用外部函数outer的局部变量arg，它保存了outer的arg值。这就是闭包。

## 从作用域的角度理解

把上面的例子改一下，在函数内部输出局部命名空间:

```python
def outer(arg):
    print locals()
    def inner():
        x = arg * arg
        print locals()
        print x
    return inner

>>> f = outer(2)
{'arg': 2}
>>> f()
{'x': 4, 'arg': 2}
4
>>>
```

在outer中，局部命名空间只有一个键arg，而在inner内部也有arg。上面说的inner保存了outer的arg值，我们就可以理解为inner的局部命名空间保存了arg。

这么一理解，就会发现闭包其实就这么简单。

另外再瞎扯一句，python中的装饰器其实就是闭包。

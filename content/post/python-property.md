+++
date = "2017-03-03T19:10:15+08:00"
description = ""
title = "Python Week 0001 --- property"
thumbnail = ""
tags = ["Python"]
categories = ["Python EveryWeek"]

+++


博客搭了有好久了，都是断断续续的写，甚至大半年没有新的内容。已经掌握的或者是新学习的，没有记录一旦长时间不用就会忘掉，毕竟还没有形成本能。所以还是想记录一些东西吧。哈哈，其实还有个原因就是写了没人看哈，然后就没动力了。

这个系列打算一周更新一篇。定位就是内容不一定多，尽量把东西讲清楚吧，按照为什么-是什么-怎么用的进行描述。

那我们就开始吧，这篇主要是关于property。

<!--more-->

### 问题引出

我们一般对属性的的操作主要有2个，访问和修改。看个例子。

```python
class Person(object):
    def __init__(self, name):
        self.name = name

p = Person('Jack')
print p.name
p.name = 'Rose'
print p.name

```

我们叫这种使用属性的方式叫点模式（我自己取得。。。），那么问题来了，如果我们想要修改属性的时候加入一个简单的类型检查，应该怎么做？

很容易想到新增一个辅助方法来达成目标。

```python
class Person(object):
    def __init__(self, name):
        self.name = name
    
    def set_name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self.name = value
        
p = Person('Jack')
p.set_name('Rose')
```

我们也看到了，这种方法虽然可以实现功能，但是使用起来不是很方便。所以我们会想有没有一种方法可以做到使用点模式的时候也能够做类型检查。

### 解决方法

我们可以使用内置的property装饰器来实现。

```python
class Person(object):

    def __init__(self, name):
        self._name = name

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        if not isinstance(value, basestring):
            raise TypeError('Expected a string')
        self._name = value

    @name.deleter
    def name(self):
        del self._name
        
p = Person('Jack')
p.name = 'Rose'
p.name = 222
del p.name
```

```shell
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-4-40579c6671bf> in <module>()
      1 p = Person('Jack')
      2 p.name = 'Rose'
----> 3 p.name = 222

<ipython-input-3-335cb8c0ab8b> in name(self, value)
     14     def name(self, value):
     15         if not isinstance(value, str):
---> 16             raise TypeError('Expected a string')
     17         self._name = value
     18 

TypeError: Expected a string
```

用起来真是so easy啊。

> 虽然对外访问是name属性，但是内部有个实际存放值的属性_name。

另外还有一种用法。

```python
class Person(object):

    def __init__(self, name):
        self._name = name

    def get_name(self):
        return self._name

    def set_name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._name = value

    def del_name(self):
        del self._name

    name = property(get_name, set_name, del_name)
```



我们看看官方的代码注释，一目了然。

```python
"""
property(fget=None, fset=None, fdel=None, doc=None) -> property attribute
        
fget is a function to be used for getting an attribute value, and likewise
fset is a function for setting, and fdel a function for del'ing, an
attribute.  Typical use is to define a managed attribute x:
        
class C(object):
    def getx(self): return self._x
    def setx(self, value): self._x = value
    def delx(self): del self._x
    x = property(getx, setx, delx, "I'm the 'x' property.")
        
Decorators make defining new properties or modifying existing ones easy:
        
class C(object):
    @property
    def x(self):
        "I am the 'x' property."
        return self._x
    @x.setter
    def x(self, value):
        self._x = value
    @x.deleter
    def x(self):
        del self._x
"""
```



以后当我们遇到设置属性的时候需要额外逻辑的时候，就可以考虑使用property了。
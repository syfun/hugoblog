+++
tags = ["Python"]
categories = ["Python"]
date = "2015-07-27T22:25:34+08:00"
description = ""
title = "Python装饰器的几种类型"
thumbnail = ""

+++


装饰器的原理就是闭包，这在[前面](/post/python-bibao/)已经提到过了。本篇主要记录一下装饰器的几种类型。

## 无参数装饰器

```python
def deco(func):
    def _deco(*args, **kwargs):
        print 'call deco'
        func(*args, **kwargs)
    return _deco

@deco
def test():
    print 'call test'

# 等同于
def test():
    print 'call test'
test = deco(func)
```

<!--more-->

## 有参数装饰器

```python
def deco(*args, **kwargs):
    def _deco(func):
        print args, kwargs
        def __deco(*args, **kwargs):
            print 'call deco'
            func(*args, **kwargs)
        return __deco
    return _deco

@deco('hello', x='nihao')
def test():
    print 'call test'

# 等同于
def test():
    print 'call test'
test = deco('hello', x='nihao')(test)
```

## 类装饰器

```python
def deco(*args, **kwargs):
    def _deco(cls):
        cls.x = 12
        return cls
    return _deco

@deco('hello')
class A(object):
    pass

>>> A.x
12

# 等同于
class A(object):
    pass
A = deco('hello')(A)
```

## 装饰器类

类作为装饰器，分为有参数和无参数。同时，需要装饰的是类方法时，需要用到__get__。

### 无参数

```python
class Deco(object):
    def __init__(self, func):
        self.func = func

    def __call__(self, *args, **kwargs):
        print 'call Deco'
        self.func(*args, **kwargs)

@Deco
def test():
    print 'call test'

# 等同于
test = Deco(test)
```

### 有参数

```python
class Deco(object):
    def __init__(self, *args, **kwargs):
        print args, kwargs

    def __call__(self, func):
        def _deco(*args, **kwargs):
            print 'call Deco'
            func(*args, **kwargs)
        return _deco

@Deco('hello')
def test():
    print 'call test'

# 等同于
test = Deco('hello')(func)
```

### 装饰类方法

#### 无参数

```python
class Deco(object):
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        def _deco(*args, **kwargs):
            print 'call Deco'
            self.func(instance, *args, **kwargs)
        return _deco

class A(object):
    @Deco
    def test(self):
        print 'call test'

# 等同于
class A(object):

    def test(self):
        print 'call test'
    test = Deco(test)
```

#### 有参数

```python
class Deco(object):
    def __init__(self, *args, **kwargs):
        print args, kwargs

    def __get__(self, instance, owner):
        def _deco(*args, **kwargs):
            print 'call Deco'
            self.func(instance, *args, **kwargs)
        return _deco

    def __call__(self, func):
        self.func = func
        return self

class A(object):

    @Deco('hello')
    def test(self):
        print 'call test'

# 等同于
class A(object):

    def test(self):
        print 'call test'
    test = Deco('hello')(test)
```

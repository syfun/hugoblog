+++
categories = ["Python"]
date = "2015-08-08T22:27:41+08:00"
description = ""
title = "Pythons HOWTO之属性描述符"
thumbnail = ""
tags = ["Python"]

+++


今天开个新坑，Pythons HOWTOS系列，主要是对官方文档的翻译。由于英语水平有限，基本上都是意译。这里附上[链接](https://docs.python.org/2.7/howto/index.html)。

本篇是第一篇，主要说的是[属性描述符](https://docs.python.org/2.7/howto/descriptor.html#id2)。

<!--more-->

## 摘要

本篇的内容主要是定义属性描述符(descriptor)，概述一下描述符协议的内容。通过自定义的的一个描述符和Python内建的描述符(functions, properties, static methods, class methods)来演示属性描述符是如何调用的。同时会给出相同功能的Python实现代码和一个简单的程序。

属性描述符不仅给一个大的工具集(暂时没发现是什么)提供了接口，它还能加深理解Python的工作原理和优雅的设计思想。

## 定义和介绍

一般来说，一个描述符是一个有“绑定行为”的对象属性，这个属性访问被描述符协议中的方法所覆盖。这些方法是\_\_get\_\_()，\_\_set\_\_()和\_\_delete\_\_()。如果某个对象定义了其中一个，那么这个对象就可以被叫做描述符。

访问属性默认通过get，set或者是delete来操作对象属性字典来实现。例如，a.x有一个查找队列，从a.\_\_dict\_\_['x']开始，然后是type(a).\_\_dict\_\_['x']，接着是type(a)的基类(metaclass除外)，以此类推。如果查找的是一个定义了描述符方法的对象，那么Python会覆盖默认行为而去调用描述符方法。发生在优先级队列的哪个位置取决于定义的描述符方法。注意，属性描述符只适用于新式类(从object或者typ继承的类)。

属性描述符是一个强大的通用协议。它是properties, methods, static methods, class methods 和super()的调用原理。它贯穿整个Python，并且用来实现2.2版本中引进的新式类。属性描述符简化了底层的C代码，还为日常Python编程提供了新的工具集。


## 描述符协议

```python
descr.__get__(self, obj, type=None) --> value

descr.__set__(self, obj, value) --> None

descr.__delete__(self, obj) --> None
```

上面的三个方法就是协议的全部内容了。定义其中任意一个方法的对象就被称为属性描述符，能够覆盖默认的属性查找规则。

如果一个对象同时定义了\_\_get\_\_和\_\_set\_\_方法，它被称做数据描述符(data descriptor)。只定义\_\_get\_\_方法的对象则被称为非数据描述符(non-data descriptor，一般用在函数方法上，其他用法也是可能的)。

数据和非数据描述符的区别在于如果某个实例属性字典中有项和描述符同名，那么属性访问的优先级是不同的。数据描述符的优先级比实例字典中项的高，非数据描述符则相反。

举个例子说明一下优先级问题:

```python
class DataDesc(object):

    def __init__(self, name=None):
        self.name = name
        self.value = None

    def __get__(self, obj, type=None):
        return self.value

    def __set__(self, obj, value):
        self.value = value


class NonDataDesc(object):

    def __init__(self, name=None):
        self.name = name
        self.value = None

    def __get__(self, obj, type=None):
        return self.value


class DataTest(object):
    x = DataDesc()


class NonDataTest(object):
    x = NonDataDesc()

>>> d = DataTest()
>>> nd = NonDataTest()
>>> d.__dict__['x'] = 2
>>> nd.__dict__['x'] = 2
>>> print d.__dict__, nd.__dict__
{'x': 2} {'x': 2}
>>> print d.x, nd.x
None 2
```

如果想要构造一个只读的数据描述符，同时定义\_\_get\_\_和\_\_set\_\_方法，并且\_\_set\_\_调用时引发一个AtrributeError异常。

## 属性描述符调用

一个属性描述符可以通过它的方法名直接调用。比如，d.\_\_get\_\_(obj)。更常见的方式是通过属性访问自动调用。比如，obj.d在obj的字典中查找d。如果d定义了\_\_get\_\_()，那么根据下文将要提到的优先级规则，d.\_\_get\_\_(obj)将会被调用。

调用的细节由obj是对象还是类来决定。

对于对象，访问是调用object.\_\_getattribute\_\_()，其中将b.x转换成type(b).\_\_dict\_\_['x'].\_\_get\_\_(b, type(b))。在实现中，数据描述符优先级最高，依次是实例变量，非数据描述符，最后是\_\_getattr\_\_()(如果定义了)。C实现能够在[Objects/object.c](https://hg.python.org/cpython/file/2.7/Objects/object.c)中的[PyObject\_GenericGetAttr()](https://docs.python.org/2.7/c-api/object.html#c.PyObject_GenericGetAttr)找到。

对于类，访问是调用type.\_\_getattribute\_\_()，其中将B.x转换成B.\_\_['x'].\_\_get\_\_(None, B)。如果用Python实现，它是这样的：

```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
       return v.__get__(None, self)
    return v
```

需要记住下面几个重要的点:

1. 描述符通过\_\_getattribute\_\_()被调用
2. 重写\_\_getattribute\_\_()能够改变自动的调用
3. \_\_getattribute\_\_()只适用于新式类
4. object.\_\_getattribute\_\_()和type.\_\_getattribute\_\_()调用\_\_get\_\_()的方式不同
5. 数据描述符总是覆盖实例字典
6. 非数据描述符可能被实例字典覆盖


super()返回的对象有一个自定义的\_\_getattribute\_\_()。调用super(B, obj).m()在obj.\_\_class\_\_.\_\_mro\_\_查找到紧跟在B后面的基类A，然后返回A.\_\_dict\_\_['m'].\_\_get\_\_(obj, B)。如果不是一个描述符，m被原封不动的返回。如果不在字典中，m转而去调用object.\_\_getattribute\_\_()查找。


注意，在Python2.2中，运行super(B, obj).m()时，如果m是一个数据描述符，将会只调用\_\_get\_\_()。在Python2.3中，除了是旧式类，非数据描述符也会得到调用。具体实现在[Objects/typeobject.c](https://hg.python.org/cpython/file/2.7/Objects/typeobject.c)的super_getattro()中。

综上所述，描述符机制嵌入到了object、type和super()的\_\_getattribute\_\_()方法中。如果类需要这个机制，必须继承自object或者是有metaclass提供类似的功能。同样的，也可以通过重写\_\_getattribute\_\_()来改变属性描述符。

## 属性描述符示例

下面的代码创建了一个类，它的实例对象是数据描述符，get和set方法中都打印了一条信息。重写\_\_getattribute\_\_()方法也可以做到这个。但是，使用描述符对监控一些属性很有用：


```python
class RevealAccess(object):
    """A data descriptor that sets and returns values
       normally and prints a message logging their access.
    """

    def __init__(self, initval=None, name='var'):
        self.val = initval
        self.name = name

    def __get__(self, obj, objtype):
        print 'Retrieving', self.name
        return self.val

    def __set__(self, obj, val):
        print 'Updating', self.name
        self.val = val

>>> class MyClass(object):
    x = RevealAccess(10, 'var "x"')
    y = 5

>>> m = MyClass()
>>> m.x
Retrieving var "x"
10
>>> m.x = 20
Updating var "x"
>>> m.x
Retrieving var "x"
20
>>> m.y
5
```

## Properties

使用property()能够把数据描述符变成属性调用。形式如下:

```python
property(fget=None, fset=None, fdel=None, doc=None) -> property attribute
```

一个典型的用法：

```python
class C(object):
    def getx(self): return self.__x
    def setx(self, value): self.__x = value
    def delx(self): del self.__x
    x = property(getx, setx, delx, "I'm the 'x' property.")
```

也可以使用装饰器:

```python
class C(object):
  @property
  def x(self):
    return self.__x

  @x.setter
  def setx(self, value):
    self.__x = value

  @x.deleter
  del delx(self):
    self.__x

>>> c = C()
>>> c.x = 2
>>> c.x
2
>>> del c.x
```

proptery()是C实现的，我们这里给出Python版本的等价实现：

```python
class Property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"

    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

一个电子表格类可能通过Cell('b10').value访问某个单元，后面希望改进成每次访问都重新计算。但是开发者不想直接改变现有的属性访问代码。那么便可以用proptery数据描述符封装属性访问。

```python
class Cell(object):
    . . .
    def getvalue(self, obj):
        "Recalculate cell before returning value"
        self.recalc()
        return obj._value
    value = property(getvalue)
```

## Functions and methods

Python的面向对象特征是建立在基于函数的环境上。使用非数据描述符，两者能够无缝融合。

类字典中用函数(function)形式存储方法(method)。在类的定义中，用def和lambda定义方法，这也是定义函数的方式。方法和普通函数唯一的区别是方法的第一个参数预留给了对象实例。按照Python的惯例，实例引用一般用self表示，当然也有可能用this或者其他变量表示。

为了支持方法调用，函数中包含了\_\_get\_\_()属性。这意味着，所有的函数都是非数据描述符。对象和类的方法，\_\_get\_\_()返回值是不同的，分别绑定(bound)和非绑定(unbound)方法。如果用纯Python表示，可能是这样的:

```python
class Function(object):
    . . .
    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        return types.MethodType(self, obj, objtype)
```

```python
>>> class D(object):
     def f(self, x):
          return x

>>> d = D()
>>> D.__dict__['f'] # Stored internally as a function
<function f at 0x00C45070>
>>> D.f             # Get from a class becomes an unbound method
<unbound method D.f>
>>> d.f             # Get from an instance becomes a bound method
<bound method D.f of <__main__.D object at 0x00B18C90>>
```

bound和unbound方法是两个不同的类型。C实现中只是一个相同的对象的两种不同表现，区别就在于im_self被设置了或者是NULL值，具体实现位于[Objects/classobject.c](https://hg.python.org/cpython/file/2.7/Objects/classobject.c)中[PyMethod_Type](https://docs.python.org/2.7/c-api/method.html#c.PyMethod_Type)。

同样地，调用方法时有没有im_self效果是不同的。如果被设置了，表明是bound方法，原始函数(存在im_func中)被调用，当然第一个参数被设置成对象实例。如果是unbound方法，所有参数原封不动地传给原始函数。C实现instancemethod_call()会更加复杂，因为有很多类型检测。

## Static methods and class methods

函数有\_\_get\_\_()属性，所以当它们被当成属性访问时会被转变成方法。非数据描述符将obj.f(\*args)转换成了f(obj, \*args)。调用klass.f(\*args)变成了f(\*args)。

下面这个表格总结了这转变方式，以及两个变种staticmethod和classmethod。

| Transformation | Called from an Object | Called from a Class |
| :------------- | :-------------------- | :------------------ |
| function       | f(obj, \*args)        | f(\*args)           |
| staticmethod   | f(\*args)             | f(\*args)           |
| classmethod    | f(type(obj), \*args)  | f(klass, \*args)    |

静态方法没有对函数做任何改变。调用c.f等价于object.\_\_getattribute\_\_(c, "f")，调C.f等于object.\_\_getattribute\_\_(C, "f")。所以，对象和类对静态方法的调用方式是统一的。静态方法不需要self。

```python
>>> class E(object):
     def f(x):
          print x
     f = staticmethod(f)

>>> print E.f(3)
3
>>> print E().f(3)
3
```

纯Pythond的staticmethod()实现可能是这样的:

```python
class StaticMethod(object):
 "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

 def __init__(self, f):
      self.f = f

 def __get__(self, obj, objtype=None):
      return self.f
```

而类方法的第一个参数是类的引用。也是分为对象调用和类调用。

```python
>>> class E(object):
     def f(klass, x):
          return klass.__name__, x
     f = classmethod(f)

>>> print E.f(3)
('E', 3)
>>> print E().f(3)
('E', 3)
```

如果函数只需要类引用而不关心底层的数据，那么类方法就会很有用。一个使用classmethod的例子是创建类构造器。在Python2.3中dict.fromkeys()从关键字列表中创建一个新的字典。纯Python可能是这样的：

```python
class Dict(object):
    . . .
    def fromkeys(klass, iterable, value=None):
        "Emulate dict_fromkeys() in Objects/dictobject.c"
        d = klass()
        for key in iterable:
            d[key] = value
        return d
    fromkeys = classmethod(fromkeys)

>>> Dict.fromkeys('abracadabra')
{'a': None, 'r': None, 'b': None, 'c': None, 'd': None}
```

classmethod的纯Python实现可能是这样的：

```python
class ClassMethod(object):
     "Emulate PyClassMethod_Type() in Objects/funcobject.c"

     def __init__(self, f):
          self.f = f

     def __get__(self, obj, klass=None):
          if klass is None:
               klass = type(obj)
          def newfunc(*args):
               return self.f(klass, *args)
          return newfunc
```


终于结束了，这篇断断续续的翻译了好几天，几次都想放弃了，但还是忍着翻译了下来，算是收获了许多。学习是没有捷径的。

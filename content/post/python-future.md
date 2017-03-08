+++
thumbnail = ""
tags = ["Python"]
categories = ["Python"]
date = "2015-07-03T22:18:29+08:00"
description = ""
title = "Python __future__ 模块"

+++


在Python2.7代码中经常能看到使用\_\_future\_\_模块。那么\_\_future\_\_到底是做什么的呢？

## 简介

从单词含义上猜应该是“未来”的模块。它有下面几个[目的](https://docs.python.org/2.7/library/__future__.html)：

1. 避免和现有分析import工具混淆，并得到你期望的模块
2. 确保2.1之前的版本导入\_\_future\_\_产生运行时异常，因为2.1之前没有这个模块
3. 文档化不兼容的改变，通常这些改变会在新版中强制执行。这类文档以可执行的形式组织，通过导入\_\_future\_\_进行可编程式的检查。

以上是对官方解释的粗略翻译，翻译起来感觉有些拗口。我是这么理解的，某个版本中出现了某个新的功能特性，而且这个特性和当前版本中使用的不兼容，也就是它在该版本中不是语言标准，那么我如果想要使用的话就需要从\_\_future\_\_模块导入。在2.1版本之前并没有\_\_future\_\_，所以使用它会引发异常。当然，在以后的某个版本中，比如说3中，某个特性已经成为标准的一部分，那么使用该特性就不用从\_\_future\_\_导入了。


下面说一下\_\_future\_\_是如何实现新特性的。

<!--more-->

## _Feature类

\_\_future\_\_.py中有形如下面的语句：

```python
FeatureName = _Feature(OptionalRelease, MandatoryRelease, CompilerFlag)

class _Feature:
    def __init__(self, optionalRelease, mandatoryRelease, compiler_flag):
        self.optional = optionalRelease    # 某个特性被认可的初始版本
        self.mandatory = mandatoryRelease  # 某个特性成为标准的版本
        self.compiler_flag = compiler_flag

    def getOptionalRelease(self):
        """Return first release in which this feature was recognized.

        This is a 5-tuple, of the same form as sys.version_info.
        """

        return self.optional

    def getMandatoryRelease(self):
        """Return release in which this feature will become mandatory.

        This is a 5-tuple, of the same form as sys.version_info, or, if
        the feature was dropped, is None.
        """

        return self.mandatory

    def __repr__(self):
        return "_Feature" + repr((self.optional,
                                  self.mandatory,
                                  self.compiler_flag))
```

## OptionalRelease参数

通常OptionalRelease版本小于MandatoryRelease，每个都是5个元素的元组，类似sys.version_info。

```python
(PY_MAJOR_VERSION, # the 2 in 2.1.0a3; an int
PY_MINOR_VERSION, # the 1; an int
PY_MICRO_VERSION, # the 0; an int
PY_RELEASE_LEVEL, # "alpha", "beta", "candidate" or "final"; string
PY_RELEASE_SERIAL # the 3; an int
)

# 例如: (2, 1, 0, "alpha", 3)表示2.1.0a3版
```

OptionalRelease版本中开始通过下列方式使用某个特性：

```python
from __future__ import FeatureName
```

## MandatoryRelease参数

在MandatoryRelease版本中该特性变成Python标准的一部分。此外MandatoryRelease版本后不需要上面的导入语句就能使用该特性。MandatoryRelease可能是None，表示一个计划中的特性被放弃了。

## CompilerFlag参数

CompilerFlag编译器标志，它是一个位域标志，传给内建函数compile()做第四个参数，用来在动态编译代码的时候允许新的特性。

CompilerFlag值等价于Include/compile.h的预定义的CO_xxx标志。

## Python2 \_\_future\_\_模块的features

一共是以下7种，其对应的CompilerFlag:

```python
all_feature_names = [
    "nested_scopes",
    "generators",
    "division",
    "absolute_import",
    "with_statement",
    "print_function",
    "unicode_literals",
]

CO_NESTED            = 0x0010   # nested_scopes
CO_GENERATOR_ALLOWED = 0        # generators (obsolete, was 0x1000)
CO_FUTURE_DIVISION   = 0x2000   # division
CO_FUTURE_ABSOLUTE_IMPORT = 0x4000 # perform absolute imports by default
CO_FUTURE_WITH_STATEMENT  = 0x8000   # with statement
CO_FUTURE_PRINT_FUNCTION  = 0x10000   # print function
CO_FUTURE_UNICODE_LITERALS = 0x20000 # unicode string literals
```

### nested_scopes

```python
nested_scopes = _Feature((2, 1, 0, "beta",  1),
                         (2, 2, 0, "alpha", 0),
                         CO_NESTED)
```

nested_scopes指的是嵌套作用域，2.1.0b1中出现，2.2.0a0中成为标准。

提到作用域，那么就不得不说命名空间。

#### 命名空间的定义

Python命名空间是名称到对象的映射，目前是用字典实现，键名是变量名，值是变量的值。比如：

```python
>>> x = 3
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__name__': '__main__', '__doc__': None, 'x': 3, '__package__': None}
```

可以看到变量x，3以字典的形式存放在globals空间内。以之对应的名称空间还有：locals()。

```python
>>> locals()
{'__builtins__': <module '__builtin__' (built-in)>, '__name__': '__main__', '__doc__': None, 'x': 3, '__package__'
```

实际上，你可以通过向名字添加键名和值，然后就可以直接使用名称了：

```python
>>> globals()['y'] = 5
>>> y
5
```

通常，我们用属性来称呼'.'点号之后的名称为属性。比如，在z.real中，real是z的一个属性。严格来说，模块中的名称引用就是属性引用。modname.funcname中，modname是一个模块对象，而funcname是它的一个属性。模块属性和全局名称有映射关系，它们共享全局命名空间。上面代码中的x和y都是模块__main__的属性。

属性可能是只读的，也可能是可写的。模块属性是可写的，你可以这么做，modname.the_answer = 42。可写的属性能够用del语句来删除。比如，del modname.the_answer将会从模块modname中删除the_answer属性。

```python
>>> x = 3
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x029FA270>, 'x': 3, '__name__': '__main__', '__doc__': None}
>>> del x
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x029FA270>, '__name__': '__main__', '__doc__': None}
```

del做的事实际上是删除了全局名称字典中的x键值。

#### 命名空间的种类

Python中有三种命名空间：

a) 局部，函数内的命名空间就是局部的，它记录了函数的变量，包括函数的参数和局部定义的变量。

b) 全局，模块内的命名空间就是全局的，它记录了模块的变量，包括函数、类、其它导入的模块、模块级的变量和常量。

c) 内置，包括异常类型、内建函数和特殊方法，可以代码中任意地方调用；

上面提到的globals是全局命名空间，locals是局部命名空间。

命名空间会在不同的时间创建，并有不同的生命周期。包含内置名称的命名空间是在python解释器启动的时候创建的，并且不会被删除。一个模块的全局命名空间是在模块定义代码被读入的时候创建的，一般情况下，模块命名空间会持续到解释器结束。在解释器最上层调用的代码，不管是从脚本中读入的还是在交互式界面中，都会被认为是属于一个叫做__main__模块的,所以它们有自己的全局命名空间。（内置名称实际上也放置在一个模块中，称为builtins）。

一个函数的局部命名空间在函数被调用的时候创建，在函数返回或者引发一个不在函数内部处理的异常时被删除。（实际上用遗忘来描述这个删除比较好。）当然了，递归调用的函数每个都有它们自己的命名空间。


#### 命名空间的可见性（作用域）

作用域是一个Python程序中命名空间直接可见的代码区域，也就是说这个区域内可以直接使用命名空间内的名称。

a) 内置命名空间在代码所有位置都是可见的，所以可以随时被调用；

b) 全局命名空间和局部命名空间中， 如果有同名变量，在全局命名空间处，局部命名空间内的同名变量是不可见的；

c) 在局部命名空间处，全局命名空间的同名变量是不可见的（只有变量不同名的情况下，可使用 global关键字让其可见）。


#### 命名空间的查找顺序

a) 如果在函数内调用一个变量，先在函数内（局部命名空间）查找，如果找到则停止查找。否则在函数外部（全局命名空间）查找，如果还是没找到，则查找内置命名空间。如果以上三个命名都未找到，则抛出NameError 的异常错误。

b) 如果在函数外调用一个变量，则在函数外查找（全局命名空间，局部命名空间此时不可见），如果找到则停止查找，否则到内置命名空间中查找。如果两者都找不到，则抛出异常。只有当局部命名空间内，使用global 关键字声明了一个变量时，查找顺序则是 a) 的查找顺序。

#### nested_scopes说明

Python2.2引入了一种略有不同但重要的改变，它会影响命名空间的搜索顺序：嵌套的作用域。在2.0中，当你在一个嵌套函数或 lambda 函数中引用一个变量时，Python会在当前（嵌套的或 lambda）函数的名称空间中搜索，然后在模块的名称空间。2.2将支在当前（嵌套的或 lambda）函数的名称空间中搜索，然后是在父函数的名称空间，接着是模块的名称空间。2.1可以两种方式工作，缺省地，按n2.0的方式工作，如果想像2.2中那样工作，使用下面的导入语句：

```python
from __future__ import nested_scopes
```

当然现在一般都用2.7或者3了，所以已经是嵌套作用域了。

来看下面一段代码：

```python
>>> x = 3
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x028BA2B0>, 'x': 3, '__name__': '__main__', '__doc__': None}
>>> def func():
        x = 2
	      print locals()

>>> func()
{'x': 2}
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x028BA2B0>, 'x': 3, '__name__': '__main__', '__doc__': None}
```

全局命名空间中x的值前后并没有改变，反而在func函数的局部命名空间中产生了一个新的名称x。由此可以看出，外层作用域的命名空间对于内层来说是只读的，当写一个同名的名称时，只会在内层生成一个新的名称。但是如果一个名称被声明为global，对其引用和复制都会直接作用域全局名称。

```python
>>> x = 2
>>> def func():
        global x
	      x = 3
	      print locals()

>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x029FA270>, 'x': 2, '__name__': '__main__', '__doc__': None
>>> func()
{}
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'func': <function func at 0x029FA270>, 'x': 3, '__name__': '__main__', '__doc__': None}
```

x的值前后改变了，而且func函数中也没用增加x。

#### import module和from module import func

import module将模块自身导入到当前命名空间，所以如果要使用module的某个函数或属性，只能module.func这么用。

而使用from module import func，则是将函数func导入当前的名称空间，这时候使用这个函数就不需要模块名称而是直接使用func。

我们通过一段代码来描述：

```python
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, '__name__': '__main__', '__doc__': None}
>>> import os
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, '__name__': '__main__', 'os': <module 'os' from 'C:\Python27\lib\os.pyc'>, '__doc__': None}
>>> del os
>>> from os import sys
>>> globals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'sys': <module 'sys' (built-in)>, '__name__': '__main__', '__doc__': None}
```

是不是很清晰，担任sys也是一个模块，如果要使用sys模块的属性，也必须要使用sys模块名了。这也是嵌套作用域的一个例子。


额。。。貌似本文的正题是\_\_future\_\_，哈哈，扯远了，我们继续来看下面一个feauture。

### generators

```python
generators = _Feature((2, 2, 0, "alpha", 1),
                      (2, 3, 0, "final", 0),
                      CO_GENERATOR_ALLOWED)
```

generators生成器起于2.2.0a1版，在2.3.0f0中成为标准。

#### 简介

生成器是这样一个函数，它记住上一次返回时在函数体中的位置。对生成器函数的第二次（或第 n 次）调用跳转至该函数中间，而上次调用的所有局部变量都保持不变。Python中yeild就是一个生成器。

#### yield 生成器的运行机制

当你问生成器要一个数时，生成器会执行，直至出现 yield 语句，生成器把 yield 的参数给你，之后生成器就不会往下继续运行。 当你问他要下一个数时，他会从上次的状态。开始运行，直至出现yield语句，把参数给你，之后停下。如此反复直至退出函数。

#### 示例

```python
>>> def my_generator():
	      yield 1
	      yield 2
	      yield 3

>>> gen = my_generator()
>>> gen.next()
1
>>> gen.next()
2
>>> gen.next()
3
>>> gen.next()

Traceback (most recent call last):
  File "<pyshell#92>", line 1, in <module>
    gen.next()
StopIteration
>>> for n in my_generator:
        print n
1
2
3
```

yield在被next调用之前并没有执行（for循环内部也是使用next），在执行完最后一个yield之后再继续调用next，那么就好遇到StopIteration异常了。这里涉及到迭代器了，不再进行详细的描述了，后面会单开一章来讲Python的三大器：迭代器、生成器、装饰器。

### division

```python
division = _Feature((2, 2, 0, "alpha", 2),
                    (3, 0, 0, "alpha", 0),
                    CO_FUTURE_DIVISION)
```

这个很简单，举例说明一下大家就懂了。

```python
# python2.7中，不导入__future__
>>> 10/3
3

# 导入__future__
>>> from __future__ import division
>>> 10/3
3.3333333333333335
```

很容易看出来，2.7中默认的整数除法是结果向下取整，而导入了\_\_future\_\_之后除法就是真正的除法了。这也是python2和python3的一个重要区别。

### absolute_import

```python
absolute_import = _Feature((2, 5, 0, "alpha", 1),
                           (3, 0, 0, "alpha", 0),
                           CO_FUTURE_ABSOLUTE_IMPORT)
```

python2.7中，在默认情况下，导入模块是相对导入的（relative import），比如说

```python
from . import json
from .json import json_dump
```

这些以'.'点导入的是相对导入，而绝对导入（absolute import）则是指从系统路径sys.path最底层的模块导入。比如:

```python
import os
from os import sys
```

### with_statement

```python
with_statement = _Feature((2, 5, 0, "alpha", 1),
                          (2, 6, 0, "alpha", 0),
                          CO_FUTURE_WITH_STATEMENT)
```

with语句也不详细讲了，看这篇[浅谈Python的with语句](http://blog.syfun.net/2015/07/07/python-with-statement)

### print_function

```python
print_function = _Feature((2, 6, 0, "alpha", 2),
                          (3, 0, 0, "alpha", 0),
                          CO_FUTURE_PRINT_FUNCTION)
```

这个就是最经典的python2和python3的区别了，python2中print不需要括号，而在python3中则需要。

```python
# python2.7
print "Hello world"

# python3
print("Hello world")
```

### unicode_literals

```python
unicode_literals = _Feature((2, 6, 0, "alpha", 2),
                            (3, 0, 0, "alpha", 0),
                            CO_FUTURE_UNICODE_LITERALS)
```

这是unicode的问题，讲起来又是长篇大论了，容我偷个懒，后面再讲吧。


至此，\_\_future\_\_模块的几个特性，算是说完了。好多内容都是参照官方文档，所以大家还是多看文档吧。

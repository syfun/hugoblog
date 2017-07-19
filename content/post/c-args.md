+++
date = "2015-06-17T22:01:25+08:00"
description = ""
title = "C语言可变参数"
thumbnail = ""
tags = ["其他"]

+++

在python中写一个有可变参数的函数或者方法是很容易的，比如下面这个例子:

```python
def print_args(*args, **kwargs):
	print type(args), args
	print type(kwargs), kwargs
	for arg in args:
		print arg
	for arg, value in kwargs.items():
		print '%s = %s' % (arg, value)

>>> print_args(1, 2, 3, x=4, y=5)
<type 'tuple'> (1, 2, 3)
<type 'dict'> {'y': 5, 'x': 4}
1
2
3
y = 5
x = 4

```
<!--more-->

从打印的信息中可以看出，args是一个元组，而kwargs是一个字典。

可以说在python中操作可变参数是非常简单的，那么C语言中是否同样有类似的功能？

答案是肯定的。我们都该知道printf函数的参数的个数是可变的。

```c
#include <stdio.h>
int printf(const char *format, ...);


printf("Hello,world!");  //其参数个数为1个。
printf("a=%d,b=%s,c=%c", a, b, c);  //其参数个数为4个。
```

printf原型中...表示参数个数是不定的。那么我们该怎么实现这个可变参数函数呢？

为了编写可变参数函数，我们通常需要用到<stdarg.h>头文件中定义的以下函数:

```c
void va_start(va_list ap, last);
type va_arg(va_list ap, type);
void va_end(va_list ap);
void va_copy(va_list dest, va_list src);
```

其中：
va_list是用于存放参数列表的数据结构。

va_start函数根据初始化last来初始化参数列表。

va_arg函数用于从参数列表中取出一个参数，参数类型由type指定。

va_copy函数用于复制参数列表。

va_end函数执行清理参数列表的工作。

上述函数通常用宏来实现，例如标准ANSI形式下，这些宏的定义是：

```c
typedef char * va_list; //字符串指针
//字节对齐
#define _INTSIZEOF(n) ( (sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1) )
#define va_start(ap,v) ( ap = (va_list)&v + _INTSIZEOF(v) )
#define va_arg(ap,t) ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) )
#define va_end(ap) ( ap = (va_list)0 )
```
可以参考一下:
http://www.cnblogs.com/wangyonghui/archive/2010/07/12/1776068.html
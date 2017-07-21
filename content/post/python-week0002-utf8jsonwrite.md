+++
categories = ["Python EveryWeek"]
date = "2016-10-30T16:03:12+08:00"
description = ""
tags = ["Python"]
thumbnail = ""
title = "Python Week 0002 --- Write UTF-8 Json Data to File"
+++

### Question

最近遇到个问题，在Mongo导出的json文件里, 用编辑器打开中文是可以正常显示的。但是我自己直接写入文件中却是"\u4f60"这样的形式。

```python
import json

d = {'你好': 'Python3'}

with open('out.json', 'w') as f:
    f.write(json.dumps(d))

with open('out.json', 'r') as f:
    print(f.read())
```

```shell
{"\u4f60\u597d": "Python3"}
```

最后打印的值并不是中文。

<!--more-->

### Method

我猜测有两处可能有两处原因。

1. json dumps返回的值不包含中文
2. 写入文件的时候转换成了ASCII形式

针对第一点，我们来测试一下。

```python
import json

d = {'你好': 'Python3'}
b = json.dumps(d)
print(b)
```

```shell
{"\u4f60\u597d": "Python3"}
```

咦，这个时候值已经变成了ASCII形式。猜想是不是`dumps`的时候做了转换。这样的话我们需要看一下`dumps`的实现。

```python
def dumps(obj, *, skipkeys=False, ensure_ascii=True, check_circular=True,
        allow_nan=True, cls=None, indent=None, separators=None,
        default=None, sort_keys=False, **kw):

If ``ensure_ascii`` is false, then the return value can contain non-ASCII
characters if they appear in strings contained in ``obj``. Otherwise, all
such characters are escaped in JSON strings. 
```

如果```ensure_ascii```是false，返回值才能包含非ASCII字符。

```python
import json

d = {'你好': 'Python3'}

with open('out.json', 'w') as f:
    f.write(json.dumps(d, ensure_ascii=False))

with open('out.json', 'r') as f:
    print(f.read())
```

```shell
{'你好': 'Python3'}
```

Perfect! 看来这就是问题所在。

> 上面猜测的第二点并没有什么问题。

### Extension

其实，除了`dumps`之外，写入文件我们还可以用更简单的`dump`方法，同样需要```ensure_ascii=False```。

```python
import json

d = {'你好': 'Python3'}

with open('out.json', 'w') as f:
    json.dump(d, f, ensure_ascii=False)

with open('out.json', 'r') as f:
    print(f.read())
```

```shell
{'你好': 'Python3'}
```
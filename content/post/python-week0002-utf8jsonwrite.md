+++
categories = ["Python EveryWeek"]
date = "2017-03-11T16:03:12+08:00"
description = ""
tags = ["Python"]
thumbnail = ""
title = "Python Week 0002 --- Write UTF-8 Json Data to File"
draft = true
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

<!--more-->






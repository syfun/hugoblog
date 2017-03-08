+++
thumbnail = ""
tags = []
categories = ["云计算"]
date = "2016-01-04T19:07:44+08:00"
description = ""
title = "Openstack Swift tempurl 和 largeobject 支持"

+++


## Tempurl

在使用网盘时，我们有时会把文件共享给没有账户权限的人，这时候就需要tempurl中间件的支持。tempurl中间件可以生成一个有时限的GET链接，其他人只需要这个链接就可以进行下载。

<!--more-->

#### URL格式

我们先看下面的例子(偷懒了，这是官方提供的)：

```
https://swift-cluster.example.com/v1/my_account/container/object
?temp_url_sig=da39a3ee5e6b4b0d3255bfef95601890afd80709
&temp_url_expires=1323479485
&filename=My+Test+File.pdf
```

这个URL包括了下面几个部分：

1. 对象路径
2. temp_url_sig，用HMAC-SHA1生成的签名，使用HTTP方法、过期时间、对象路径和一个秘钥生成
3. temp_url_expires，过期时间
4. 文件名，这是可选的，覆盖默认的对象名

#### 秘钥

秘钥主要指的是账户和容器的一个秘钥属性。

```
账户：
X-Account-Meta-Temp-URL-Key
X-Account-Meta-Temp-URL-Key-2

容器：
X-Container-Meta-Temp-URL-Key
X-Container-Meta-Temp-URL-Key-2
```

通过如下方式设置：

```sh
账户：
swift post -m "Temp-URL-Key:MYKEY"

容器：
swift post container -m "Temp-URL-Key:MYKEY"
```
#### HMAC-SHA1签名

同样，先给例子：

```python
import hmac
from hashlib import sha1
from time import time
method = 'GET'
duration_in_seconds = 60*60*24
expires = int(time() + duration_in_seconds)
path = '/v1/my_account/container/object'
key = 'MYKEY'
hmac_body = '%s\n%s\n%s' % (method, expires, path)
signature = hmac.new(key, hmac_body, sha1).hexdigest()
```

method也可以是PUT方法。

#### swift tempurl命令

先看例子：

```
swift tempurl GET 3600 /v1/my_account/container/object MYKEY
```

help信息：

```sh
Generates a temporary URL for a Swift object.

Positional arguments:
  <method>              An HTTP method to allow for this temporary URL.
                        Usually 'GET' or 'PUT'.
  <seconds>             The amount of time in seconds the temporary URL will be
                        valid for; or, if --absolute is passed, the Unix
                        timestamp when the temporary URL will expire.
  <path>                The full path to the Swift object. Example:
                        /v1/AUTH_account/c/o.
  <key>                 The secret temporary URL key set on the Swift cluster.
                        To set a key, run 'swift post -m
                        "Temp-URL-Key:b3968d0207b54ece87cccc06515a89d4"'

Optional arguments:
  --absolute            Interpet the <seconds> positional argument as a Unix
                        timestamp rather than a number of seconds in the
                        future.
```

## Static Large object

manifest文件，分片按顺序存
```
[
    {
        "path": "mycontainer/objseg1",
        "etag": "0228c7926b8b642dfb29554cd1f00963",
        "size_bytes": 1468006
    },
    {
        "path": "mycontainer/pseudodir/seg-obj2",
        "etag": "5bfc9ea51a00b790717eeb934fb77b9b",
        "size_bytes": 1572864
    },
    {
        "path": "other-container/seg-final",
        "etag": "b9c3da507d2557c1ddc51f27c54bae51",
        "size_bytes": 256
    }
]
```

将manifest上传，添加?multipart-manifest=put。上传完成后，X-Static-Large-Object为true。

下载正常。

删除要删除分片的话需要?multipart-manifest=delete

COPY时候?multipart-manifest=get只拷贝manifest。

断点续传：
```
http://192.168.99.100:8080/v1/AUTH_a5b3c5a9f5354e4da74e9d0af7b849a9/test?prefix=ss&format=json
```

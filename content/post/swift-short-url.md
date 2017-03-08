+++
categories = ["Python"]
date = "2014-06-10T19:22:40+08:00"
description = ""
title = "url短网址算法python实现"
thumbnail = ""
tags = []

+++



最近在做swift对象存储的共享功能，由于共享url过长，就想到有没有一种缩短的算法。网上查了下，便找到这种url短网址算法。

算法的基本思路如下：

 1. 短网址一般有6位，假如每一位是由[a-z0-9A-Z]共62个字符组成，一共有62^6~=568亿种组合。
 2. 将长url存入数据库，用返回的ID转换成6位短url，再将该短url存入数据库，这样就有长短url的映射关系了。
 
 <!--more-->

数据结构：

 - id，int类型，自动增长
 - lurl，string类型
 - surl，string类型

那么如何转换ID呢？

[How to code a URL shortener?][1]告诉我们一种不错的方法，借鉴进制之间的转换。id是个十进制的值，上述62个字符可以组成一个62进制，这样便可以轻松的转换了。转换算法如下:

```python
def base62(value_10):
    value_62 = []
    while 1:
        if value_10 < 62:
            value_62.append(value_10)
            break
        else:
            remainder = value_10 % 62
            value_62.append(remainder)
            value_10 = value_10 / 62
    zero_num = 6 - len(value_62)
    value_62.extend([0]*zero_num)
    return value_62

def base10(value_62):
    value_10 = 0
    for i in range(len(value_62)):
        value_10 += (value_62[i]*62**i)
    return value_10
```
上述代码实现10进制id和了62进制的转换，之后找到对应的字符便能组成一个新的url，代码如下：

```python
def shorturl(lurl):
    charlist = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i',
                'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r',
                's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '0',
                '1', '2', '3', '4', '5', '6', '7', '8', '9',
                'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I',
                'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R',
                'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z']
    id = db_add(lurl)
    value_62 = base62(id)
    surl = []
    for i in value_62:
        surl.append(charlist[i])
    surl = ''.join(surl)
    db_save(id, surl)
    return surl
```
在实际应用中还要再考虑2个问题，62个字符的顺序以及用什么数据库来存储url。

打乱62个字符的顺序可以用洗牌算法，思路如下：

 1. 生成一个1到62之间的随机数，将1和这个随机数位置的字符调换位置
 2. 生成一个2到62之间的随机数，将2和这个随机数位置的字符调换位置
 3. 以此类推，第i次时，生成一个i到62的随机数，调换位置

```python
def shuffle(n):
    sl = []
    for i in range(n):
        sl.append(i)

    for i in range(n):
        rn = randint(i, n-1)
        sl[i], sl[rn] = sl[rn], sl[i]

    return sl
```

至于用什么数据库，个人推荐TTserver(Tokyo Cabinet)，它是一款 DBM 数据库，该数据库读写非常快，哈希模式写入100万条数据只需0.643秒，读取100万条数据只需0.773秒，是 Berkeley DB 等 DBM 的几倍。

  [1]: http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener

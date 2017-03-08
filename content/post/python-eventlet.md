+++
thumbnail = ""
tags = []
categories = ["Python"]
date = "2014-10-15T19:13:20+08:00"
description = ""
title = "eventlet的设计模式"
draft = true
+++

最近看openstack的源码，到处看到eventlet，所以就想学习下，其实也就是阅读下[官方的文档](http://eventlet.net/doc/)。

eventlet一共有3种设计模式，client、server、dispatch。

###Client Pattern
一个典型的client端例子就是web crawler（网络爬虫）。
```python
import eventlet
from eventlet.green import urllib2

urls = ["http://www.google.com/intl/en_ALL/images/logo.gif",
       "https://wiki.secondlife.com/w/images/secondlife.jpg",
       "http://us.i1.yimg.com/us.yimg.com/i/ww/beta/y3.gif"]

def fetch(url):
    return urllib2.urlopen(url).read()

pool = eventlet.GreenPool()
for body in pool.imap(fetch, urls):
    print("got body", len(body))
```

<!--more-->

`from eventlet.green import urllib2`导入eventlet.green中的urllib2模块，它类似标准的urllib2模块，只是通信部分用的是green sockets。<br/>
`pool = eventlet.GreenPool()`实例了一个拥有1000个线程的GreenPool(默认是1000，可以在实例化的时候通过size参数改变)。为什么要用GreenPool？因为它能大大增加网络爬虫同时抓取的url数目。<br/>
`for body in pool.imap(fetch, urls)`，迭代多个并行fetch函数的执行结果。`imap`能够并行调用函数，并且按序返回它们执行的结果。<br/>
client模式的关键是它涉及到收集每个函数调用的结果。最终表现是这些函数并发执行。注意，imap会有一个内存上限，所以当url的数目达到上万时，并不会消耗GB级别的内存。

###Server Pattern
下面是server端的例子，一个简单的echo server:
```python
import eventlet

def handle(client):
    while True:
        c = client.recv(1)
        if not c: break
        client.sendall(c)

server = eventlet.listen(('0.0.0.0', 6000))
pool = eventlet.GreenPool(10000)
while True:
    new_sock, address = server.accept()
    pool.spawn_n(handle, new_sock)
```

`server = eventlet.listen(('0.0.0.0', 6000))`，创建监听socket很方便。<br/>
`pool = eventlet.GreenPool(10000)`创建了一个能够处理10000个客户端的线程池。<br/>
`pool.spawn_n(handle, new_sock)`，一个线程处理一个客户端。接收循环不关心handle函数的结果，所以这里用spawn_n代替spawn。spawn_n无返回值，spawn返回一个GreenThread对象。

server模式和client模式之间的区别是server模式有一个while循环去重复的调用accept()，而且将client端的socket完全交给handle()去处理，而不是自己去收集结果。

###Dispatch Pattern
还有一个常用的模式是dispatch（调度）模式。它是一个server同时也是其他server的client。下面是一个例子，一个server接收POST请求，这个请求包含RSS feeds（消息源）的urls列表。server并发取出所有的消息源，返回它们的标题给client。
```python
import eventlet
feedparser = eventlet.import_patched('feedparser')

pool = eventlet.GreenPool()

def fetch_title(url):
    d = feedparser.parse(url)
    return d.feed.get('title', '')

def app(environ, start_response):
    pile = eventlet.GreenPile(pool)
    for url in environ['wsgi.input'].readlines():
        pile.spawn(fetch_title, url)
    titles = '\n'.join(pile)
    start_response('200 OK', [('Content-type', 'text/plain')])
    return [titles]
```
这个例子中用GreenPool的全局变量去控制并发。如果外部请求的数目没有全局限制，那么一个client可能导致server打开上万个并发连接，从而导致feedparser的IP被禁止，或者其他一些严重的问题。这个线程池不是一个完全的Dos攻击保护，但是起码有一点作用。

`pile = eventlet.GreenPile(pool)`用全局pool来实例化GreenPile，pile中保存各个线程的运行结果。

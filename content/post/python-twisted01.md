+++
date = "2015-07-14T22:22:03+08:00"
description = ""
title = "Twisted源码分析系列01-reactor"
thumbnail = ""
tags = ["Python"]
categories = ["Python"]

+++


## 简介

Twisted是用Python实现的事件驱动的网络框架。

如果想看教程的话，我觉得写得最好的就是[Twisted Introduction](http://krondo.com/?page_id=1327)了，这是[翻译](https://github.com/syfun/twisted-intro-cn)。

<!--more-->

下面就直接进入主题了。

我们通过一个示例开始分析源码，那么先看下面这个示例。

```python
#!/usr/bin/env python
# coding=utf8

from twisted.internet.protocol import Protocol, ServerFactory


HOST = '127.0.0.1'
PORT = 8080


class EchoProtocol(Protocol):

    def dataReceived(self, data):
        self.transport.write(data)


if __name__ == '__main__':
    factory = ServerFactory()
    factory.protocol = EchoProtocol

    from twisted.internet import reactor
    reactor.listenTCP(PORT, factory, interface=HOST)
    reactor.run()
```

这是一个非常简单的Echo server，每当有数据发来，都会将数据往回发。

## reactor

reactor是事件管理器，用于注册、注销事件，运行事件循环，当事件发生时调用回调函数处理。关于reactor有下面几个结论:

1. Twisted的reactor只有通过调用reactor.run()来启动。
2. reactor循环是在其开始的进程中运行，也就是运行在主进程中。
3. 一旦启动，就会一直运行下去。reactor就会在程序的控制下（或者具体在一个启动它的线程的控制下）。
4. reactor循环并不会消耗任何CPU的资源。
5. 并不需要显式的创建reactor，只需要引入就OK了。

最后一条需要解释清楚。在Twisted中，reactor是Singleton（也就是单例模式），即在一个程序中只能有一个reactor，并且只要你引入它就相应地创建一个。上面引入的方式这是twisted默认使用的方法，当然了，twisted还有其它可以引入reactor的方法。例如，可以使用twisted.internet.pollreactor中的系统调用来poll来代替select方法。
若使用其它的reactor，需要在引入twisted.internet.reactor前安装它。下面是安装pollreactor的方法：

```python
from twisted.internet import pollreactor
pollreactor.install()
```

如果你没有安装其它特殊的reactor而引入了twisted.internet.reactor，那么Twisted会根据操作系统安装默认的reactor。正因为如此，习惯性做法不要在最顶层的模块内引入reactor以避免安装默认reactor，而是在你要使用reactor的区域内安装。
下面是使用 pollreactor重写上上面的程序:tho

```python
from twited.internet import pollreactor
pollreactor.install()
from twisted.internet import reactor
reactor.run()
```

那么reactor是如何实现单例的？来看一下from twisted.internet import reactor做了哪些事情就并明白了。

下面是twisted/internet/reactor.py的部分代码:

```python
# twisted/internet/reactor.py
import sys
del sys.modules['twisted.internet.reactor']
from twisted.internet import default
default.install()
```

注：Python中所有加载到内存的模块都放在sys.modules，它是一个全局字典。当import一个模块时首先会在这个列表中查找是否已经加载了此模块，如果加载了则只是将模块的名字加入到正在调用import的模块的命名空间中。如果没有加载则从sys.path目录中按照模块名称查找模块文件，找到后将模块载入内存，并加入到sys.modules中，并将名称导入到当前的命名空间中。

假如我们是第一次运行from twisted.internet import reactor，因为sys.modules中还没有twisted.internet.reactor，所以会运行reactory.py中的代码，安装默认的reactor。之后，如果导入的话，因为sys.modules中已存在该模块，所以会直接将sys.modules中的twisted.internet.reactor导入到当前命名空间。

default中的install：

```python
# twisted/internet/default.py
def _getInstallFunction(platform):
    """
    Return a function to install the reactor most suited for the given platform.

    @param platform: The platform for which to select a reactor.
    @type platform: L{twisted.python.runtime.Platform}

    @return: A zero-argument callable which will install the selected
        reactor.
    """
    try:
        if platform.isLinux():
            try:
                from twisted.internet.epollreactor import install
            except ImportError:
                from twisted.internet.pollreactor import install
        elif platform.getType() == 'posix' and not platform.isMacOSX():
            from twisted.internet.pollreactor import install
        else:
            from twisted.internet.selectreactor import install
    except ImportError:
        from twisted.internet.selectreactor import install
    return install


install = _getInstallFunction(platform)
```

很明显，default中会根据平台获取相应的install。Linux下会首先使用epollreactor，如果内核还不支持，就只能使用pollreactor。Mac平台使用pollreactor，windows使用selectreactor。每种install的实现差不多，这里我们抽取selectreactor中的install来看看。



```python
# twisted/internet/selectreactor.py：
def install():
    """Configure the twisted mainloop to be run using the select() reactor.
    """
    # 单例
    reactor = SelectReactor()
    from twisted.internet.main import installReactor
    installReactor(reactor)

# twisted/internet/main.py：
def installReactor(reactor):
    """
    Install reactor C{reactor}.

    @param reactor: An object that provides one or more IReactor* interfaces.
    """
    # this stuff should be common to all reactors.
    import twisted.internet
    import sys
    if 'twisted.internet.reactor' in sys.modules:
        raise error.ReactorAlreadyInstalledError("reactor already installed")
    twisted.internet.reactor = reactor
    sys.modules['twisted.internet.reactor'] = reactor
```

在installReactor中，向sys.modules添加twisted.internet.reactor键，值就是再install中创建的单例reactor。以后要使用reactor，就会导入这个单例了。

## SelectReactor

```python
# twisted/internet/selectreactor.py
@implementer(IReactorFDSet)
class SelectReactor(posixbase.PosixReactorBase, _extraBase)
```

implementer表示SelectReactor实现了IReactorFDSet接口的方法，这里用到了[zope.interface](http://docs.zope.org/zope.interface/)，它是python中的接口实现，有兴趣的同学可以去看下。

IReactorFDSet接口主要对描述符的获取、添加、删除等操作的方法。这些方法看名字就能知道意思，所以我就没有加注释。

```python
# twisted/internet/interfaces.py
class IReactorFDSet(Interface):

    def addReader(reader):

    def addWriter(writer):

    def removeReader(reader):

    def removeWriter(writer):

    def removeAll():

    def getReaders():

    def getWriters():

```

### reactor.listenTCP()

示例中的reactor.listenTCP()注册了一个监听事件，它是父类PosixReactorBase中方法。

```python
# twisted/internet/posixbase.py
@implementer(IReactorTCP, IReactorUDP, IReactorMulticast)
class PosixReactorBase(_SignalReactorMixin, _DisconnectSelectableMixin,
                       ReactorBase):

    def listenTCP(self, port, factory, backlog=50, interface=''):
        p = tcp.Port(port, factory, backlog, interface, self)
        p.startListening()
        return p

# twisted/internet/tcp.py
@implementer(interfaces.IListeningPort)
class Port(base.BasePort, _SocketCloser):
    def __init__(self, port, factory, backlog=50, interface='', reactor=None):
       """Initialize with a numeric port to listen on.
       """
       base.BasePort.__init__(self, reactor=reactor)
       self.port = port
       self.factory = factory
       self.backlog = backlog
       if abstract.isIPv6Address(interface):
           self.addressFamily = socket.AF_INET6
           self._addressType = address.IPv6Address
       self.interface = interface
    ...

    def startListening(self):
       """Create and bind my socket, and begin listening on it.
          创建并绑定套接字，开始监听。

       This is called on unserialization, and must be called after creating a
       server to begin listening on the specified port.
       """
       if self._preexistingSocket is None:
           # Create a new socket and make it listen
           try:
               # 创建套接字
               skt = self.createInternetSocket()
               if self.addressFamily == socket.AF_INET6:
                   addr = _resolveIPv6(self.interface, self.port)
               else:
                   addr = (self.interface, self.port)
               # 绑定
               skt.bind(addr)
           except socket.error as le:
               raise CannotListenError(self.interface, self.port, le)
           # 监听
           skt.listen(self.backlog)
       else:
           # Re-use the externally specified socket
           skt = self._preexistingSocket
           self._preexistingSocket = None
           # Avoid shutting it down at the end.
           self._shouldShutdown = False

       # Make sure that if we listened on port 0, we update that to
       # reflect what the OS actually assigned us.
       self._realPortNumber = skt.getsockname()[1]

       log.msg("%s starting on %s" % (
               self._getLogPrefix(self.factory), self._realPortNumber))

       # The order of the next 5 lines is kind of bizarre.  If no one
       # can explain it, perhaps we should re-arrange them.
       self.factory.doStart()
       self.connected = True
       self.socket = skt
       self.fileno = self.socket.fileno
       self.numberAccepts = 100

       # startReading调用reactor的addReader方法将Port加入读集合
       self.startReading()
```

整个逻辑很简单，和正常的server端一样，创建套接字、绑定、监听。不同的是将套接字的描述符添加到了reactor的读集合。那么假如有了client连接过来的话，reactor会监控到，然后触发事件处理程序。

## reacotr.run()事件主循环

```python
# twisted/internet/posixbase.py
@implementer(IReactorTCP, IReactorUDP, IReactorMulticast)
class PosixReactorBase(_SignalReactorMixin, _DisconnectSelectableMixin,
                       ReactorBase)

# twisted/internet/base.py
class _SignalReactorMixin(object):

    def startRunning(self, installSignalHandlers=True):
        """
        PosixReactorBase的父类_SignalReactorMixin和ReactorBase都有该函数，但是
        _SignalReactorMixin在前，安装mro顺序的话，会先调用_SignalReactorMixin中的。
        """
        self._installSignalHandlers = installSignalHandlers
        ReactorBase.startRunning(self)

    def run(self, installSignalHandlers=True):
        self.startRunning(installSignalHandlers=installSignalHandlers)
        self.mainLoop()

    def mainLoop(self):
        while self._started:
            try:
                while self._started:
                    # Advance simulation time in delayed event
                    # processors.
                    self.runUntilCurrent()
                    t2 = self.timeout()
                    t = self.running and t2
                    # doIteration是关键，select,poll,epool实现各有不同
                    self.doIteration(t)
            except:
                log.msg("Unexpected error in main loop.")
                log.err()
            else:
                log.msg('Main loop terminated.')
```

mianLoop就是最终的主循环了，在循环中，调用doIteration方法监控读写描述符的集合，一旦发现有描述符准备好读写，就会调用相应的事件处理程序。

```python
# twisted/internet/selectreactor.py
@implementer(IReactorFDSet)
class SelectReactor(posixbase.PosixReactorBase, _extraBase):

    def __init__(self):
        """
        Initialize file descriptor tracking dictionaries and the base class.
        """
        self._reads = set()
        self._writes = set()
        posixbase.PosixReactorBase.__init__(self)

    def doSelect(self, timeout):
        """
        Run one iteration of the I/O monitor loop.

        This will run all selectables who had input or output readiness
        waiting for them.
        """
        try:
            # 调用select方法监控读写集合，返回准备好读写的描述符
            r, w, ignored = _select(self._reads,
                                    self._writes,
                                    [], timeout)
        except ValueError:
            # Possibly a file descriptor has gone negative?
            self._preenDescriptors()
            return
        except TypeError:
            # Something *totally* invalid (object w/o fileno, non-integral
            # result) was passed
            log.err()
            self._preenDescriptors()
            return
        except (select.error, socket.error, IOError) as se:
            # select(2) encountered an error, perhaps while calling the fileno()
            # method of a socket.  (Python 2.6 socket.error is an IOError
            # subclass, but on Python 2.5 and earlier it is not.)
            if se.args[0] in (0, 2):
                # windows does this if it got an empty list
                if (not self._reads) and (not self._writes):
                    return
                else:
                    raise
            elif se.args[0] == EINTR:
                return
            elif se.args[0] == EBADF:
                self._preenDescriptors()
                return
            else:
                # OK, I really don't know what's going on.  Blow up.
                raise

        _drdw = self._doReadOrWrite
        _logrun = log.callWithLogger
        for selectables, method, fdset in ((r, "doRead", self._reads),
                                           (w,"doWrite", self._writes)):
            for selectable in selectables:
                # if this was disconnected in another thread, kill it.
                # ^^^^ --- what the !@#*?  serious!  -exarkun
                if selectable not in fdset:
                    continue
                # This for pausing input when we're not ready for more.

                # 调用_doReadOrWrite方法
                _logrun(selectable, _drdw, selectable, method)

    doIteration = doSelect

    def _doReadOrWrite(self, selectable, method):
        try:
            # 调用method，doRead或者是doWrite，
            # 这里的selectable可能是我们监听的tcp.Port
            why = getattr(selectable, method)()
        except:
            why = sys.exc_info()[1]
            log.err()
        if why:
            self._disconnectSelectable(selectable, why, method=="doRead")
```

那么假如客户端有连接请求了，就会调用读集合中tcp.Port的doRead方法。

```python
# twisted/internet/tcp.py

@implementer(interfaces.IListeningPort)
class Port(base.BasePort, _SocketCloser):

    def doRead(self):
        """Called when my socket is ready for reading.
        当套接字准备好读的时候调用

        This accepts a connection and calls self.protocol() to handle the
        wire-level protocol.
        """
        try:
            if platformType == "posix":
                numAccepts = self.numberAccepts
            else:
                numAccepts = 1
            for i in range(numAccepts):
                if self.disconnecting:
                    return
                try:
                    # 调用accept
                    skt, addr = self.socket.accept()
                except socket.error as e:
                    if e.args[0] in (EWOULDBLOCK, EAGAIN):
                        self.numberAccepts = i
                        break
                    elif e.args[0] == EPERM:
                        continue
                    elif e.args[0] in (EMFILE, ENOBUFS, ENFILE, ENOMEM, ECONNABORTED):
                        log.msg("Could not accept new connection (%s)" % (
                            errorcode[e.args[0]],))
                        break
                    raise

                fdesc._setCloseOnExec(skt.fileno())
                protocol = self.factory.buildProtocol(self._buildAddr(addr))
                if protocol is None:
                    skt.close()
                    continue
                s = self.sessionno
                self.sessionno = s+1
                # transport初始化的过程中，会将自身假如到reactor的读集合中，那么当它准备
                # 好读的时候，就可以调用它的doRead方法读取客户端发过来的数据了
                transport = self.transport(skt, protocol, addr, self, s, self.reactor)
                protocol.makeConnection(transport)
            else:
                self.numberAccepts = self.numberAccepts+20
        except:
            log.deferr()
```

doRead方法中，调用accept产生了用于接收客户端数据的套接字，将套接字与transport绑定，然后把transport加入到reactor的读集合。当客户端有数据到来时，就会调用transport的doRead方法进行数据读取了。

Connection是Server（transport实例的类）的父类，它实现了doRead方法。

```python
# twisted/internet/tcp.py
@implementer(interfaces.ITCPTransport, interfaces.ISystemHandle)
class Connection(_TLSConnectionMixin, abstract.FileDescriptor, _SocketCloser,
                 _AbortingMixin):

    def doRead(self):
        try:
            # 接收数据
            data = self.socket.recv(self.bufferSize)
        except socket.error as se:
            if se.args[0] == EWOULDBLOCK:
                return
            else:
                return main.CONNECTION_LOST

        return self._dataReceived(data)

    def _dataReceived(self, data):
        if not data:
            return main.CONNECTION_DONE
        # 调用我们自定义protocol的dataReceived方法处理数据
        rval = self.protocol.dataReceived(data)
        if rval is not None:
            offender = self.protocol.dataReceived
            warningFormat = (
                'Returning a value other than None from %(fqpn)s is '
                'deprecated since %(version)s.')
            warningString = deprecate.getDeprecationWarningString(
                offender, versions.Version('Twisted', 11, 0, 0),
                format=warningFormat)
            deprecate.warnAboutFunction(offender, warningString)
        return rval
```

_dataReceived中调用了示例中我们自定义的EchoProtocol的dataReceived方法处理数据。

至此，一个简单的流程，从创建监听事件，到接收客户端数据就此结束了。

一些细节的地方我并未说明，这里只是说明一个大概的流程，想看的细一点的，可以直接跟着这个流程去看源码。

这个系列应该会不定时更新，如果有人感兴趣的，也可以和我直接交流。


---
title: tornado -- ioloop源码解析
description: tornado
categories:
 - tornado
tags:
 - tornado
---


## Ioloop

> ioloop 核心的I/O循环
Tornado为了实现高并发和高性能，使用了一个IOLoop事件循环来处理socket的读写事件，IOLoop事件循环是基于Linux的epoll模型，可以高效地响应网络事件，这是Tornado高效的基础保证。

## 目标

更好的理解ioloop的事件循环。


- 对应调用来进行阅读，首先找一份ioloop调用代码。

    以下为ioloop源码中提供的代码，功能是一个tcp的服务端

``` python
import errno
import functools
import ioloop
import socket

def connection_ready(sock, fd, events):
    while True:
        try:
            connection, address = sock.accept()
            print connection, address
        except socket.error, e:
            if e[0] not in (errno.EWOULDBLOCK, errno.EAGAIN):
                raise
            return
        connection.setblocking(0)
        # handle_connection(connection, address)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setblocking(0)
sock.bind(('0.0.0.0', 8888))
sock.listen(128)

# 单例对象
io_loop = ioloop.IOLoop.instance()
# 偏函数 上篇文章有讲道
callback = functools.partial(connection_ready, sock)
# 添加处理对应
io_loop.add_handler(sock.fileno(), callback, io_loop.READ)
# 开始事件循环
io_loop.start()
```
- ioloop.IOLoop.instance()

    对用源码
    ``` python
        @classmethod
        def instance(cls):
        # 单例
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance
    ```
    一个经典的单例模式

- io_loop.add_handler(sock.fileno(), callback, io_loop.READ)

    对应源码
    ``` python
        # 注册处理方法到对应的描述符上
        def add_handler(self, fd, handler, events):
            """Registers the given handler to receive the given events for fd."""
            self._handlers[fd] = handler
            self._impl.register(fd, events | self.ERROR)
    ```
    看起来比较简单的逻辑，将描述符和对应处理方法相绑定

    self._impl.register

    实际调用的epoll添加事件的方法

    epoll.epoll_ctl(self._epoll_fd, self._EPOLL_CTL_ADD, fd, events)

- io_loop.start()
    实现循环的核心逻辑
    ```python
        def start(self):
        self._running = True
        while True:
            # Never use an infinite timeout here - it can stall epoll
            poll_timeout = 0.2

            # Prevent IO event starvation by delaying new callbacks
            # to the next iteration of the event loop.
            callbacks = list(self._callbacks)
            for callback in callbacks:
                # A callback can add or remove other callbacks
                if callback in self._callbacks:
                    self._callbacks.remove(callback)
                    self._run_callback(callback)

            if self._callbacks:
                poll_timeout = 0.0

            if self._timeouts:
                now = time.time()
                while self._timeouts and self._timeouts[0].deadline <= now:
                    timeout = self._timeouts.pop(0)
                    self._run_callback(timeout.callback)
                if self._timeouts:
                    milliseconds = self._timeouts[0].deadline - now
                    poll_timeout = min(milliseconds, poll_timeout)

            if not self._running:
                break

            try:
                # 获取可读写事件
                event_pairs = self._impl.poll(poll_timeout)
            except Exception, e:
                if e.args == (4, "Interrupted system call"):
                    logging.warning("Interrupted system call", exc_info=1)
                    continue
                else:
                    raise

            # Pop one fd at a time from the set of pending fds and run
            # its handler. Since that handler may perform actions on
            # other file descriptors, there may be reentrant calls to
            # this IOLoop that update self._events
            self._events.update(event_pairs)
            while self._events:
                # 获取一个fd以及对应事件
                fd, events = self._events.popitem()
                try:
                    # 执行事件
                    self._handlers[fd](fd, events)
                except KeyboardInterrupt:
                    raise
                except OSError, e:
                    if e[0] == errno.EPIPE:
                        # Happens when the client closes the connection
                        pass
                    else:
                        logging.error("Exception in I/O handler for fd %d",
                                      fd, exc_info=True)
                except:
                    logging.error("Exception in I/O handler for fd %d",
                                  fd, exc_info=True)
    ```

    self._impl.poll(poll_timeout)获取当前读写事件

    实际调用

    epoll.epoll_wait(self._epoll_fd, int(timeout * 1000))

    后续就比较简单查找读写事件的对应方法然后执行

## 自己实现一个ioloop

实现一个简单版本能够完成上文客户端调用

``` python
# coding=utf-8
import bisect
import errno
import fcntl
import logging
import os
import select
import time

class IOLoop(object):

    _EPOLLIN = 0x001
    _EPOLLPRI = 0x002
    _EPOLLOUT = 0x004
    _EPOLLERR = 0x008
    _EPOLLHUP = 0x010
    _EPOLLRDHUP = 0x2000
    _EPOLLONESHOT = (1 << 30)
    _EPOLLET = (1 << 31)

    # Our events map exactly to the epoll events
    NONE = 0
    READ = _EPOLLIN
    WRITE = _EPOLLOUT
    ERROR = _EPOLLERR | _EPOLLHUP | _EPOLLRDHUP

    def __init__(self, impl=None):
        self._impl = select.epoll
        # self._impl = impl or _poll()
        self._handlers = {}
        self._events = {}

        r, w = os.pipe()
        self._set_nonblocking(r)
        self._set_nonblocking(w)
        self._waker_reader = os.fdopen(r, "r", 0)
        self._waker_writer = os.fdopen(w, "w", 0)
        self.add_handler(r, self._read_waker, self.WRITE)

    @classmethod
    def instance(cls):
        # 单例
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance

    def add_handler(self, fd, handler, events):
        self._handlers[fd] = handler
        self._impl.register(fd, events | self.ERROR)

    def start(self):
        self._running = True
        while True:
            poll_timeout = 0.2
            try:
                event_pairs = self._impl.poll(poll_timeout)
            except Exception, e:
                if e.args == (4, "Interrupted system call"):
                    logging.warning("Interrupted system call", exc_info=1)
                    continue
                else:
                    raise

            self._events.update(event_pairs)
            while self._events:
                fd, events = self._events.popitem()
                try:
                    self._handlers[fd](fd, events)
                except KeyboardInterrupt:
                    raise
                except OSError, e:
                    if e[0] == errno.EPIPE:
                        # Happens when the client closes the connection
                        pass
                    else:
                        logging.error("Exception in I/O handler for fd %d",
                                      fd, exc_info=True)
                except:
                    logging.error("Exception in I/O handler for fd %d",
                                  fd, exc_info=True)


    def _read_waker(self, fd, events):
        try:
            while True:
                self._waker_reader.read()
        except IOError:
            pass

    def _set_nonblocking(self, fd):
        flags = fcntl.fcntl(fd, fcntl.F_GETFL)
        # 设置不阻塞
        fcntl.fcntl(fd, fcntl.F_SETFL, flags | os.O_NONBLOCK)
```
以上代码完成了客户端调取的全部工能

ps:select.epoll 只在py2.6以上的版本存在

测试客户端代码

``` python
import socket

client = socket.socket() 
client.connect(('0.0.0.0', 8888))

msg = 'dsds' 随便发点消息 
client.send(msg.encode("utf-8"))
data = client.recv(1024)
print("recv:>", data.decode())
```

可以直接配置以上三个文件再本地进行测试更好的理解ioloop的用法


> “世界就像是个巨大的马戏团，它让你兴奋，却让我惶恐，因为我知道散场后永远是有限温存，无限心酸。”
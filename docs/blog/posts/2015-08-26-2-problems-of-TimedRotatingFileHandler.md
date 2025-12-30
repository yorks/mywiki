---
title: TimedRotatingFileHandler 的两个问题
slug: two-problems-of-TimedRotatingFileHandler
date: 2015-08-26
tags:
  - logging
  - python
  - lock
  - problem
---



### TimedRotatingFileHandler
之前遇到多进程下面日志轮转的问题，找了好久都没有找到现有的按时间轮转的解决方案。
决定自己去实现一个，所以翻翻`logging`的`TimedRotatingFileHandler`发现其中比较不合理的问题

#### 轮转时间问题
```
        def __init__(self, filename, when='h', interval=1, backupCount=0, encoding=None, delay=False, utc=False):
            ...
            if os.path.exists(filename):
                t = os.stat(filename)[ST_MTIME]
            else:
                t = int(time.time())
            self.rolloverAt = self.computeRollover(t)

        def computeRollover(self, currentTime):
            """ ... """
            result = currentTime + self.interval
            ...
```
从上面的代码可以看出，它的轮转时间是根据启动时间(日志文件存在mtime)为起点，到达 `self.interval` （单位是s)就会进行轮转，
所以轮转日志的时候就有很大可能不是在整点/整分/，即 如果你设置了每个小时轮转，启动是在`09:31:01`，那么下一个轮转点就会是`10:31:01`，
这样导致的问题是，找`10:30`的日志，需要跑文件名中的小时为 10 的日志，找`10:32`的需要从文件名小时为11的日志里面找。
并不是我们平时需要的每到整点就轮转。

#### 多进程写问题
```
    class TimedRotatingFileHandler(BaseRotatingHandler):
```
这个类继承了`BaseRotatingHandler` 我们再看看`BaseRotatingHandler`

```
    class BaseRotatingHandler(logging.FileHandler):
```
发现`BaseRotatingHandler`里面并没有什么加锁的逻辑，我们再过去看看`logging.FileHandler`

```
    class FileHandler(StreamHandler):
```
发现`FileHandler`里面对只是对`close`关闭日志文件句柄的时候做了线程锁，并没有针对写日志的锁。依然过去看看`StreamHandler`:


```
        def flush(self):
            """
            Flushes the stream.
            """
            self.acquire()
            try:
                if self.stream and hasattr(self.stream, "flush"):
                    self.stream.flush()
            finally:
                self.release()
        def emit(self, record):
            ...
            stream.write(fs % msg)
            ...
            self.flush()
            ...
```
从上面代码可以看到，`StreamHandler`里面是把日志先`write`到系统的`cache`里面，然后再调用`flush`到磁盘，
当`flush`的时候是做了线程锁的，所以写日志是线程安全的。
但是我们并没有找到相关的文件锁或者共享全局锁，即使它写日志是线程安全，如果出现多个同样的进程，
同时写同一个日志文件，会出现怎样呢？
这个时候就可能会出现不同的进程争抢文件IO的问题，可能会出现写失败而**丢日志**的情况。

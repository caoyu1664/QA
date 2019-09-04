<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Q&A**


   * [Logging](#Logging)
      * [1 logging源码分析](#1-logging源码分析)
      * [2 TimedRotatingFileHandler零点自动分割日志文件混乱](#2-TimedRotatingFileHandler零点自动分割日志文件混乱)
         * [1 分割实现的机制](#1-分割实现的机制)
         * [2 可能出现的问题](#2-可能出现的问题)
         * [3 解决方法1：linux文件锁](#3-解决方法1：linux文件锁)
         * [4 解决方法2：更改判断机制](#4-解决方法2：更改判断机制)
         * [5 解决方法3：使用其他handler](#5-解决方法3：使用其他handler)
         * [6 解决方法4: 不使用分割机制](#6-解决方法4: 不使用分割机制)
         
<!-- markdown-toc end -->


# Logging

## 1 logging源码分析

  这里将以常规操作logging流程为主线，把logging源码串一下。
  
  整体流程：`logger.debug('debug infomations') `
  ```python
  def debug(self, msg, *args, **kwargs):
      """
      Log 'msg % args' with severity 'DEBUG'.

      To pass exception information, use the keyword argument exc_info with
      a true value, e.g.

      logger.debug("Houston, we have a %s", "thorny problem", exc_info=1)
      """
      if self.isEnabledFor(DEBUG):
          self._log(DEBUG, msg, args, **kwargs)
  ```
  
  --> `Logger._log(DEBUG, msg)`
  ```python
  def _log(self, level, msg, args, exc_info=None, extra=None):
      """
      Low-level logging routine which creates a LogRecord and then calls
      all the handlers of this logger to handle the record.
      """
      if _srcfile:
          #IronPython doesn't track Python frames, so findCaller raises an
          #exception on some versions of IronPython. We trap it here so that
          #IronPython can use logging.
          try:
              fn, lno, func = self.findCaller()
          except ValueError:
              fn, lno, func = "(unknown file)", 0, "(unknown function)"
      else:
          fn, lno, func = "(unknown file)", 0, "(unknown function)"
      if exc_info:
          if not isinstance(exc_info, tuple):
              exc_info = sys.exc_info()
      record = self.makeRecord(self.name, level, fn, lno, msg, args, exc_info, func, extra)
      self.handle(record)
  ```
  
  --> `record(args obj) = Logger.makeRecord()` 生成record对象，其中包含所有日志信息相关参数
  ```python
  def makeRecord(self, name, level, fn, lno, msg, args, exc_info, func=None, extra=None):
        """
        A factory method which can be overridden in subclasses to create
        specialized LogRecords.
        """
        rv = LogRecord(name, level, fn, lno, msg, args, exc_info, func)
        if extra is not None:
            for key in extra:
                if (key in ["message", "asctime"]) or (key in rv.__dict__):
                    raise KeyError("Attempt to overwrite %r in LogRecord" % key)
                rv.__dict__[key] = extra[key]
        return rv
  ```
  
  --> `Logger.handle(record)` record对象交给handle即将分配给一个控制器
  ```python
  def handle(self, record):
        """
        Call the handlers for the specified record.

        This method is used for unpickled records received from a socket, as
        well as those created locally. Logger-level filtering is applied.
        """
        if (not self.disabled) and self.filter(record):
            self.callHandlers(record)
  ```
  
  --> `callHandlers(record)`查找用户初始化logger对象时通过logger。addHandler(some handler)添加的控制器，若缺省则使用控制器父类Handler
  ```python
  def callHandlers(self, record):
        """
        Pass a record to all relevant handlers.

        Loop through all handlers for this logger and its parents in the
        logger hierarchy. If no handler was found, output a one-off error
        message to sys.stderr. Stop searching up the hierarchy whenever a
        logger with the "propagate" attribute set to zero is found - that
        will be the last logger whose handlers are called.
        """
        c = self
        found = 0
        while c:
            for hdlr in c.handlers:
                found = found + 1
                if record.levelno >= hdlr.level:
                    hdlr.handle(record)
            if not c.propagate:
                c = None    #break out
            else:
                c = c.parent
        if (found == 0) and raiseExceptions and not self.manager.emittedNoHandlerWarning:
            sys.stderr.write("No handlers could be found for logger"
                             " \"%s\"\n" % self.name)
            self.manager.emittedNoHandlerWarning = 1
  ```
  
  --> `logging.FileHandler()` 负责创建和清空日志文件句柄
  ```python
  class FileHandler(StreamHandler):
    """
    A handler class which writes formatted logging records to disk files.
    """
    def __init__(self, filename, mode='a', encoding=None, delay=0):
        """
        Open the specified file and use it as the stream for logging.
        """
        #keep the absolute path, otherwise derived classes which use this
        #may come a cropper when the current directory changes
        if codecs is None:
            encoding = None
        self.baseFilename = os.path.abspath(filename)
        self.mode = mode
        self.encoding = encoding
        if delay:
            #We don't open the stream, but we still need to call the
            #Handler constructor to set level, formatter, lock etc.
            Handler.__init__(self)
            self.stream = None
        else:
            StreamHandler.__init__(self, self._open())

    def close(self):
        """
        Closes the stream.
        """
        self.acquire()
        try:
            if self.stream:
                self.flush()
                if hasattr(self.stream, "close"):
                    self.stream.close()
                StreamHandler.close(self)
                self.stream = None
        finally:
            self.release()

    def _open(self):
        """
        Open the current base file with the (original) mode and encoding.
        Return the resulting stream.
        """
        if self.encoding is None:
            stream = open(self.baseFilename, self.mode)
        else:
            stream = codecs.open(self.baseFilename, self.mode, self.encoding)
        return stream

    def emit(self, record):
        """
        Emit a record.

        If the stream was not opened because 'delay' was specified in the
        constructor, open it before calling the superclass's emit.
        """
        if self.stream is None:
            self.stream = self._open()
        StreamHandler.emit(self, record)

  ```
  
  --> `StreamHandler.emit(record)`负责写入日志
  ```python
  def emit(self, record):
        """
        Emit a record.

        If a formatter is specified, it is used to format the record.
        The record is then written to the stream with a trailing newline.  If
        exception information is present, it is formatted using
        traceback.print_exception and appended to the stream.  If the stream
        has an 'encoding' attribute, it is used to determine how to do the
        output to the stream.
        """
        try:
            msg = self.format(record)
            stream = self.stream
            fs = "%s\n"
            if not _unicode: #if no unicode support...
                stream.write(fs % msg)
            else:
                try:
                    if (isinstance(msg, unicode) and
                        getattr(stream, 'encoding', None)):
                        ufs = fs.decode(stream.encoding)
                        try:
                            stream.write(ufs % msg)
                        except UnicodeEncodeError:
                            #Printing to terminals sometimes fails. For example,
                            #with an encoding of 'cp1251', the above write will
                            #work if written to a stream opened or wrapped by
                            #the codecs module, but fail when writing to a
                            #terminal even when the codepage is set to cp1251.
                            #An extra encoding step seems to be needed.
                            stream.write((ufs % msg).encode(stream.encoding))
                    else:
                        stream.write(fs % msg)
                except UnicodeError:
                    stream.write(fs % msg.encode("UTF-8"))
            self.flush()
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            self.handleError(record)
  ```
  
  ## 2 TimedRotatingFileHandler零点自动分割日志文件混乱
    问题背景：Linux + Apache + flask + logging + TimedRotatingFileHandler(when='midnight')
    问题描述：这个控制器给一个midnight参数可实现每日0点自动分割日志文件，但其对多进程并不友好，服务部署在服务器后，在0点分割文件后，两个进程分别对昨天和今天的日志文件进行写入，或存在日志丢失的情况。
 
 ### 1 分割实现的机制
    分割逻辑实现在`TimedRotatingFileHandler.doRollover()`
    
先附上源码
```python
def doRollover(self):
        """
        do a rollover; in this case, a date/time stamp is appended to the filename
        when the rollover happens.  However, you want the file to be named for the
        start of the interval, not the current time.  If there is a backup count,
        then we have to get a list of matching filenames, sort them and remove
        the one with the oldest suffix.
        """
        if self.stream:
            self.stream.close()
            self.stream = None
        # get the time that this sequence started at and make it a TimeTuple
        currentTime = int(time.time())
        dstNow = time.localtime(currentTime)[-1]
        t = self.rolloverAt - self.interval
        if self.utc:
            timeTuple = time.gmtime(t)
        else:
            timeTuple = time.localtime(t)
            dstThen = timeTuple[-1]
            if dstNow != dstThen:
                if dstNow:
                    addend = 3600
                else:
                    addend = -3600
                timeTuple = time.localtime(t + addend)
        dfn = self.baseFilename + "." + time.strftime(self.suffix, timeTuple)
        if os.path.exists(dfn):
            os.remove(dfn)
        os.rename(self.baseFilename, dfn)
        if self.backupCount > 0:
            # find the oldest log file and delete it
            #s = glob.glob(self.baseFilename + ".20*")
            #if len(s) > self.backupCount:
            #    s.sort()
            #    os.remove(s[0])
            for s in self.getFilesToDelete():
                os.remove(s)
        #print "%s -> %s" % (self.baseFilename, dfn)
        self.stream = self._open()
        newRolloverAt = self.computeRollover(currentTime)
        while newRolloverAt <= currentTime:
            newRolloverAt = newRolloverAt + self.interval
        #If DST changes and midnight or weekly rollover, adjust for this.
        if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
            dstAtRollover = time.localtime(newRolloverAt)[-1]
            if dstNow != dstAtRollover:
                if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                    addend = -3600
                else:           # DST bows out before next rollover, so we need to add an hour
                    addend = 3600
                newRolloverAt += addend
        self.rolloverAt = newRolloverAt
```

主要分割逻辑在这部分：
```python
if os.path.exists(dfn):
    os.remove(dfn)
os.rename(self.baseFilename, dfn)
self.stream = self._open()
```

上述code总结起来就4步：
1. 假设当前日志文件为 httpsrv.log(self.baseFilename) 切分后的文件名为 httpsrv.log.2019-09-03(dfn)
2. 判断 httpsrv.log.2019-09-03 (dfn)是否存在，如果存在则删除
3. 将当前的日志文件名改为 httpsrv.log.2019-09-03(dfn)
4. 重新打开日志文件句柄 httpsrv.log

### 2 可能出现的问题
    1. 一个进程切换了，其他进程还在httpsrv.log.2019-09-03句柄往里写日志
    2. 一个进程切换了，把其他进程重命名的httpsrv.log.2019-09-03直接删除
    3. （我遇到的问题）
      假设前提：
      比如两个进程同时在切割文件，进程1正执行step3，进程2刚执行完setp2，
      情景一：
      进程1完成了重命名但还没创建新的httpsrv.log。。进程2开始重命名，此时进程2会因为找不到httpsrv.log
      出错。。

      情景二：
      如果进程1已经创建了httpsrv.log，进程2会把新创建的空文件httpsrv.log重命名为 httpsrv.log.2019-09-03
      此时，进程1认为已经完成了分割，而他的句柄还指向httpsrv.log.2019-09-03。。。

      所以最后 进程1文件句柄：httpsrv.log.2019-09-03     进程2文件句柄：httpsrv.log   两个进程同时写
      
      
### 3 解决方法1：linux文件锁
    改名部分加一个linux文件锁，有一个进程获取到文件句柄，其他进程阻塞，等待执行改名后再进行日志写入。
    
主要分割逻辑部分：
```python
if os.path.exists(dfn):
    os.remove(dfn)
os.rename(self.baseFilename, dfn)
self.stream = self._open()
```

改成：
```python
import fcntl
if not os.path.exists(dfn):
    f = open(self.baseFilename, 'a')
    fcntl.lockf(f.fileno(), fcntl.LOCK_EX)
    if os.path.exists(self.baseFilename):
        os.rename(self.baseFilename, dfn)
```


### 4 解决方法2：更改判断机制
    我们可以尝试改变命名的判断条件，我们的需求是，当昨天的日志文件不存在（dfn）且未分割的日志文件存在（self.baseFilename）则对未分割的文件重命名，然后创建新的文件句柄指向self.baseFilname

思路依然是改变逻辑主体：
```python
if not os.path.exists(dfn) and os.path.exists(self.baseFilename):
    os.rename(self.baseFilename, dfn)
```


### 5 解决方法3：使用其他handler
贴个基友的解决思路https://blog.csdn.net/u011361138/article/details/86139900


### 6 解决方法4: 不使用分割机制
    <光芒敬基友> <未验证>
    我们让python只负责写日志，再专门写一个linux定时任务负责执行分割逻辑（依然重命名就好）

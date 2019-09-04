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

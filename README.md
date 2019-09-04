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
    整体流程：logger.debug('debug infomations') 
    '''
    def debug(self, msg, *args, **kwargs):
        """
        Log 'msg % args' with severity 'DEBUG'.

        To pass exception information, use the keyword argument exc_info with
        a true value, e.g.

        logger.debug("Houston, we have a %s", "thorny problem", exc_info=1)
        """
        if self.isEnabledFor(DEBUG):
            self._log(DEBUG, msg, args, **kwargs)
    '''
    --> Logger._log(DEBUG, msg)
    '''
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
    '''
    --> record(args obj) = Logger.makeRecord() 生成record对象，其中包含所有日志信息相关参数
    --> Logger.handle(record) record对象交给handle即将分配给一个控制器
    --> callHandlers(record)查找用户初始化logger对象时通过logger。addHandler(some handler)添加的控制器，若缺省则使用控制器父类Handler
    --> logging.FileHandler() 负责创建和清空日志文件句柄 
    --> StreamHandler.emit(record)负责写入日志

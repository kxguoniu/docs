# asyncio 之 base_events.py
## 基础的事件循环
### 由子类重写的方法
```python
class BaseEventLoop(events.AbstractEventLoop):
	...
    def _make_socket_transport(self, sock, protocol, waiter=None, *,
                               extra=None, server=None):
        raise NotImplementedError

    def _make_ssl_transport(self, rawsock, protocol, sslcontext, waiter=None,
                            *, server_side=False, server_hostname=None,
                            extra=None, server=None):
        raise NotImplementedError

    def _make_datagram_transport(self, sock, protocol,
                                 address=None, waiter=None, extra=None):
        raise NotImplementedError

    def _make_read_pipe_transport(self, pipe, protocol, waiter=None,
                                  extra=None):
        raise NotImplementedError

    def _make_write_pipe_transport(self, pipe, protocol, waiter=None,
                                   extra=None):
        raise NotImplementedError

    @coroutine
    def _make_subprocess_transport(self, protocol, args, shell,
                                   stdin, stdout, stderr, bufsize,
                                   extra=None, **kwargs):
        raise NotImplementedError

    def _write_to_self(self):
        raise NotImplementedError

    def _process_events(self, event_list):
        raise NotImplementedError

    def _check_closed(self):
        if self._closed:
            raise RuntimeError('Event loop is closed')
```
### 初始化
```python
    def __init__(self):
		# 取消的定时任务计数器
        self._timer_cancelled_count = 0
		# loop 的关闭状态
        self._closed = False
		# 待执行的任务队列
        self._ready = collections.deque()
		# 定时任务的列表
        self._scheduled = []
		# 默认的异常调度器
        self._default_executor = None
        self._internal_fds = 0
        # 正在运行 loop 的线程标识符
        self._thread_id = None
		# 时钟分标率
        self._clock_resolution = time.get_clock_info('monotonic').resolution
		# 异常对象
        self._exception_handler = None
		# 调试模式
        self._debug = (not sys.flags.ignore_environment
                       and bool(os.environ.get('PYTHONASYNCIODEBUG')))
        # 在调试模式下,如果某一步骤执行的时间超过此设置就会记录缓慢的回调日志
        self.slow_callback_duration = 0.1
		# 当前的句柄
        self._current_handle = None
```
### 创建一个 task
```python
    def create_task(self, coro):
        self._check_closed()
		# 把协程包装成为一个 task
        task = tasks.Task(coro, loop=self)
        if task._source_traceback:
            del task._source_traceback[-1]
        return task
```
### 检查 loop 状态
如果 loop 已经被停止,抛出异常
```python
    def _check_closed(self):
        if self._closed:
            raise RuntimeError('Event loop is closed')
```
### 一直运行直到被停止(出现一个停止异常)
```python
    def run_forever(self):
        self._check_closed()
		# 如果 loop 正在运行,抛出异常
        if self.is_running():
            raise RuntimeError('Event loop is running.')
		# 设置运行 loop 的线程id
        self._thread_id = threading.get_ident()
        try:
            while True:
                try:
                    self._run_once()
				# 出现停止异常 退出循环
                except _StopError:
                    break
        finally:
            self._thread_id = None
```
### 运行直到任务完成
```python
    def run_until_complete(self, future):
        self._check_closed()
        new_task = not isinstance(future, futures.Future)
		# 如果参数是协程则需要把它包装成 task 返回
        future = tasks.async(future, loop=self)
        if new_task:
            # 不记录异常信息
            future._log_destroy_pending = False
		# 在 future 完成后向 loop 发送一个停止异常
        future.add_done_callback(_run_until_complete_cb)
        try:
            self.run_forever()
        except:
            if new_task and future.done() and not future.cancelled():
                future.exception()
            raise
		# 删除任务完成回调
        future.remove_done_callback(_run_until_complete_cb)
        if not future.done():
            raise RuntimeError('Event loop stopped before Future completed.')

        return future.result()
```
### 停止
所有在停止调用之前添加回调函数将被调用,之后添加的回调函数将不会执行,如果再次运行 run_forever 这些调用又会执行
```python
    def stop(self):
        self.call_soon(_raise_stop_error)
```
### 关闭
关闭 loop 会清空待执行任务队列,不能关闭正在运行的 event loop
```python
    def close(self):
        if self.is_running():
            raise RuntimeError("Cannot close a running event loop")
        if self._closed:
            return
        if self._debug:
            logger.debug("Close %r", self)
        self._closed = True
        self._ready.clear()
        self._scheduled.clear()
        executor = self._default_executor
        if executor is not None:
            self._default_executor = None
            executor.shutdown(wait=False)
```
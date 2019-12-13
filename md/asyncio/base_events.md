[TOC]
## class BaseEventLoop
### 初始化
```python
def __init__(self):
	# 取消的定时执行对象的数量
	self._timer_cancelled_count = 0
	# event loop 关闭状态
	self._closed = False
	# event loop 停止状态
	self._stopping = False
	# 待执行的任务队列(handle)
	self._ready = collections.deque()
	# 定时执行的任务列表(timerhandle)
	self._scheduled = []
	# 默认的异常调度器
	self._default_executor = None
	self._internal_fds = 0
	# 线程标识
	self._thread_id = None
	# 时钟
	self._clock_resolution = time.get_clock_info('monotonic').resolution
	self._exception_handler = None
	# 设置运行模式
	self.set_debug(coroutines._is_debug_mode())
	# 在调试模式下，如果任务运行的时间超过这个时间就会记录日志
	self.slow_callback_duration = 0.1
	# 当前需要执行的 handle
	self._current_handle = None
	# 创建task的工厂
	self._task_factory = None
	self._coroutine_origin_tracking_enabled = False
	self._coroutine_origin_tracking_saved_depth = None

	# 所有异步生成器的弱引用集合
	self._asyncgens = weakref.WeakSet()
	# Set to True when `loop.shutdown_asyncgens` is called.
	self._asyncgens_shutdown_called = False
```
### def create_future
创建一个`future`对象并返回。
```python
def create_future(self):
	return futures.Future(loop=self)
```
### def create_task
创建一个task并返回。
```python
def create_task(self, coro):
	self._check_closed()
	# 如果task工厂为None，使用默认的Task类创建task对象
	if self._task_factory is None:
		task = tasks.Task(coro, loop=self)
		if task._source_traceback:
			del task._source_traceback[-1]
	# 使用工厂创建task对象
	else:
		task = self._task_factory(self, coro)
	return task
```
### def set_task_factory
设置task工厂。必须是可调用的接受两个参数loop,coro，返回一个future对象
```python
def set_task_factory(self, factory):
	if factory is not None and not callable(factory):
		raise TypeError('task factory must be a callable or None')
	self._task_factory = factory
```
### def get_task_factory
```python
def get_task_factory(self):
	return self._task_factory
```
### 传输相关，需要子类重写的
```python
    def _make_socket_transport(self, sock, protocol, waiter=None, *,
                               extra=None, server=None):
        raise NotImplementedError

    def _make_ssl_transport(
            self, rawsock, protocol, sslcontext, waiter=None,
            *, server_side=False, server_hostname=None,
            extra=None, server=None,
            ssl_handshake_timeout=None,
            call_connection_made=True):
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

    async def _make_subprocess_transport(self, protocol, args, shell,
                                         stdin, stdout, stderr, bufsize,
                                         extra=None, **kwargs):
        raise NotImplementedError

    def _write_to_self(self):
        raise NotImplementedError

    def _process_events(self, event_list):
        raise NotImplementedError
```
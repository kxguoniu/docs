[TOC]
## 摘要
基础的事件循环类实现文件(主要是循环运行，创建任务，添加运行任务相关的方法)，还有一个文件发送专用的的协议类以及一个服务器类的实现。
## class _SendfileFallbackProtocol
发送文件的协议，只有第三种发送文件的方式才会用到这个协议
### 初始化
```python
class _SendfileFallbackProtocol(protocols.Protocol):
    def __init__(self, transp):
        if not isinstance(transp, transports._FlowControlMixin):
            raise TypeError("transport should be _FlowControlMixin instance")
		# 流传输
        self._transport = transp
		# 传输现在使用的协议
        self._proto = transp.get_protocol()
		# 传输的读取数据的状态
        self._should_resume_reading = transp.is_reading()
		# 传输发送数据的状态
        self._should_resume_writing = transp._protocol_paused
		# 传输暂停从套接字中接受数据
        transp.pause_reading()
		# 设置传输的协议为发送文件专用协议
        transp.set_protocol(self)
		# 如果传输正处于暂停写入数据状态
        if self._should_resume_writing:
            self._write_ready_fut = self._transport._loop.create_future()
        else:
            self._write_ready_fut = None
```
### async def drain
判断传输写入是否准备好
```python
async def drain(self):
	if self._transport.is_closing():
		raise ConnectionError("Connection closed by peer")
	fut = self._write_ready_fut
	if fut is None:
		return
	await fut
```
### def connection_made
```python
def connection_made(self, transport):
	raise RuntimeError("Invalid state: "
					   "connection should have been established already.")

```
### def connection_lost
```python
def connection_lost(self, exc):
	if self._write_ready_fut is not None:
		if exc is None:
			self._write_ready_fut.set_exception(
				ConnectionError("Connection is closed by peer"))
		else:
			self._write_ready_fut.set_exception(exc)
	self._proto.connection_lost(exc)
```
### def pause_writing
暂停发送文件
```python
def pause_writing(self):
	if self._write_ready_fut is not None:
		return
	self._write_ready_fut = self._transport._loop.create_future()
```
### def resume_writing
继续发送文件
```python
def resume_writing(self):
	if self._write_ready_fut is None:
		return
	self._write_ready_fut.set_result(False)
	self._write_ready_fut = None
```
### def data_received
发送文件是数据接收应该被暂停
```python
def data_received(self, data):
	raise RuntimeError("Invalid state: reading should be paused")

def eof_received(self):
	raise RuntimeError("Invalid state: reading should be paused")
```
### async def restore
```python
async def restore(self):
	self._transport.set_protocol(self._proto)
	if self._should_resume_reading:
		self._transport.resume_reading()
	if self._write_ready_fut is not None:
		# Cancel the future.
		# Basically it has no effect because protocol is switched back,
		# no code should wait for it anymore.
		self._write_ready_fut.cancel()
	if self._should_resume_writing:
		self._proto.resume_writing()
```
## class Server
### 初始化
```python
class Server(events.AbstractServer):

    def __init__(self, loop, sockets, protocol_factory, ssl_context, backlog,
                 ssl_handshake_timeout):
        self._loop = loop
		# 服务器套接字列表
        self._sockets = sockets
		# 活跃的连接数
        self._active_count = 0
		# 等待服务器关闭的协程
        self._waiters = []
		# 连接协议
        self._protocol_factory = protocol_factory
        self._backlog = backlog
		# ssl上下文
        self._ssl_context = ssl_context
		# ssl 握手超时时间
        self._ssl_handshake_timeout = ssl_handshake_timeout
		# 服务器状态
        self._serving = False
		# 服务其运行是设置的future
        self._serving_forever_fut = None
```
### def _attach
连接数加一
```python
def _attach(self):
	assert self._sockets is not None
	self._active_count += 1
```
### def _detach
连接数减一
```python
def _detach(self):
	assert self._active_count > 0
	self._active_count -= 1
	if self._active_count == 0 and self._sockets is None:
		self._wakeup()
```
### def _wakeup
```python
def _wakeup(self):
	waiters = self._waiters
	self._waiters = None
	for waiter in waiters:
		if not waiter.done():
			waiter.set_result(waiter)
```
### def _start_serving
启动服务器
```python
def _start_serving(self):
	if self._serving:
		return
	self._serving = True
	for sock in self._sockets:
		sock.listen(self._backlog)
		self._loop._start_serving(
			self._protocol_factory, sock, self._ssl_context,
			self, self._backlog, self._ssl_handshake_timeout)
```
### def sockets
```python
def get_loop(self):
	return self._loop

def is_serving(self):
	return self._serving

@property
def sockets(self):
	if self._sockets is None:
		return []
	return list(self._sockets)
```
### def close
关闭服务器
```python
def close(self):
	sockets = self._sockets
	if sockets is None:
		return
	self._sockets = None

	for sock in sockets:
		self._loop._stop_serving(sock)

	self._serving = False

	if (self._serving_forever_fut is not None and
			not self._serving_forever_fut.done()):
		self._serving_forever_fut.cancel()
		self._serving_forever_fut = None

	if self._active_count == 0:
		self._wakeup()
```
### async def start_serving
```python
    async def start_serving(self):
        self._start_serving()
		# 跳过一次事件循环，这样所有的套接字都已经被注册
        await tasks.sleep(0, loop=self._loop)
```
### asyncio def serve_forever

```python
async def serve_forever(self):
	if self._serving_forever_fut is not None:
		raise RuntimeError(
			f'server {self!r} is already being awaited on serve_forever()')
	if self._sockets is None:
		raise RuntimeError(f'server {self!r} is closed')

	self._start_serving()
	self._serving_forever_fut = self._loop.create_future()

	try:
		await self._serving_forever_fut
	except futures.CancelledError:
		try:
			self.close()
			await self.wait_closed()
		finally:
			raise
	finally:
		self._serving_forever_fut = None
```
### async def wait_closed
```python
async def wait_closed(self):
	if self._sockets is None or self._waiters is None:
		return
	waiter = self._loop.create_future()
	self._waiters.append(waiter)
	await waiter
```
## class BaseEventLoop
### 初始化
```python
class BaseEventLoop(events.AbstractEventLoop):
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
### 传输相关，需要子类重写的方法
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
### def _check_closed
```python
def _check_closed(self):
	if self._closed:
		raise RuntimeError('Event loop is closed')
```
### def _asyncgen_finalizer_hook
```python
def _asyncgen_finalizer_hook(self, agen):
	self._asyncgens.discard(agen)
	if not self.is_closed():
		self.call_soon_threadsafe(self.create_task, agen.aclose())
```
### def _asyncgen_firstiter_hook
```python
def _asyncgen_firstiter_hook(self, agen):
	if self._asyncgens_shutdown_called:
		warnings.warn(
			f"asynchronous generator {agen!r} was scheduled after "
			f"loop.shutdown_asyncgens() call",
			ResourceWarning, source=self)

	self._asyncgens.add(agen)
```
### async def shutdown_asyncgens
```python
async def shutdown_asyncgens(self):
	"""Shutdown all active asynchronous generators."""
	self._asyncgens_shutdown_called = True

	if not len(self._asyncgens):
		# If Python version is <3.6 or we don't have any asynchronous
		# generators alive.
		return

	closing_agens = list(self._asyncgens)
	self._asyncgens.clear()

	results = await tasks.gather(
		*[ag.aclose() for ag in closing_agens],
		return_exceptions=True,
		loop=self)

	for result, agen in zip(results, closing_agens):
		if isinstance(result, Exception):
			self.call_exception_handler({
				'message': f'an error occurred during closing of '
						   f'asynchronous generator {agen!r}',
				'exception': result,
				'asyncgen': agen
			})
```
### def run_forever
运行`event loop`直到被停止
```python
def run_forever(self):
	# 检查循环是否被关闭
	self._check_closed()
	# 循环正在运行中
	if self.is_running():
		raise RuntimeError('This event loop is already running')
	# 如果当前线程正在运行其他的事件循环，抛出异常
	if events._get_running_loop() is not None:
		raise RuntimeError(
			'Cannot run the event loop while another loop is running')
	# 设置运行的模式
	self._set_coroutine_origin_tracking(self._debug)
	# 当前线程的id
	self._thread_id = threading.get_ident()

	# 旧的异步生成器钩子函数
	old_agen_hooks = sys.get_asyncgen_hooks()
	# 设置新的异步生成器钩子函数
	sys.set_asyncgen_hooks(firstiter=self._asyncgen_firstiter_hook,
						   finalizer=self._asyncgen_finalizer_hook)
	try:
		# 设置当前运行的loop为自己
		events._set_running_loop(self)
		# 无限循环运行此方法，直到loop被停止。
		while True:
			self._run_once()
			if self._stopping:
				break
	finally:
		# 重置停止状态
		self._stopping = False
		# 重置线程id
		self._thread_id = None
		# 取消正在运行的 loop
		events._set_running_loop(None)
		# 关闭debug模式
		self._set_coroutine_origin_tracking(False)
		# 还原系统的异步生成器钩子函数
		sys.set_asyncgen_hooks(*old_agen_hooks)
```
### def run_until_complete
运行直到`future`完成。返回`future`对象的结果或者抛出异常。
不要试图对同一个协程调用两次。那是灾难性的。
```python
def run_until_complete(self, future):
	self._check_closed()

	# 传入的参数是一个协程
	new_task = not futures.isfuture(future)
	# 确保参数是一个future对象
	future = tasks.ensure_future(future, loop=self)
	if new_task:
		# 不需要记录未执行的 task 对象
		future._log_destroy_pending = False

	# 添加回调函数，在future完成后停止 event loop
	future.add_done_callback(_run_until_complete_cb)
	# 运行 event loop
	try:
		self.run_forever()
	except:
		if new_task and future.done() and not future.cancelled():
			future.exception()
		raise
	# 删除回调函数。
	finally:
		future.remove_done_callback(_run_until_complete_cb)
	# 如果future还没有完成，抛出异常。
	if not future.done():
		raise RuntimeError('Event loop stopped before Future completed.')

	return future.result()
```
### def stop
停止事件循环不是同步执行的，而是在循环中执行。
```python
def stop(self):
	self._stopping = True
```
### def close
关闭事件循环，这将清空队列并关闭执行程序。但是只能在event loop停止的状态中关闭。
```python
def close(self):
	if self.is_running():
		raise RuntimeError("Cannot close a running event loop")
	if self._closed:
		return
	if self._debug:
		logger.debug("Close %r", self)
	# 设置关闭状态
	self._closed = True
	# 清空待执行任务队列
	self._ready.clear()
	# 清空定时执行列表
	self._scheduled.clear()
	# 关闭执行程序
	executor = self._default_executor
	if executor is not None:
		self._default_executor = None
		executor.shutdown(wait=False)
```
### def is_close
判断`loop`是否已经关闭
```python
def is_closed(self):
	return self._closed
```
### def __del__
删除loop
```python
def __del__(self):
	if not self.is_closed():
		warnings.warn(f"unclosed event loop {self!r}", ResourceWarning,
					  source=self)
		if not self.is_running():
			self.close()
```
### def is_running
判断loop是不是还在运行
```python
def is_running(self):
	return (self._thread_id is not None)
```
### def time
event loop 使用的时间线。
```python
def time(self):
	return time.monotonic()
```
### def call_later
定时调用函数,使用相对时间
```python
def call_later(self, delay, callback, *args, context=None):
	# 创建一个定时调用对象。
	timer = self.call_at(self.time() + delay, callback, *args,
						 context=context)
	if timer._source_traceback:
		del timer._source_traceback[-1]
	return timer
```
### def call_at
使用绝对时间创建定时调用函数
```python
def call_at(self, when, callback, *args, context=None):
	self._check_closed()
	# 如果是调试模式
	if self._debug:
		self._check_thread()
		self._check_callback(callback, 'call_at')
	# 创建一个定时调用对象
	timer = events.TimerHandle(when, callback, args, self, context)
	if timer._source_traceback:
		del timer._source_traceback[-1]
	# 把定时对象添加到loop的定时队列中
	heapq.heappush(self._scheduled, timer)
	# 定时对象是可调用的。
	timer._scheduled = True
	return timer
```
### def call_soon
尽快的调用传入的函数
```python
def call_soon(self, callback, *args, context=None):
	self._check_closed()
	# 调试模式
	if self._debug:
		self._check_thread()
		self._check_callback(callback, 'call_soon')
	# 创建一个调用的对象
	handle = self._call_soon(callback, args, context)
	if handle._source_traceback:
		del handle._source_traceback[-1]
	return handle
```
### def _check_callback
调试模式下检查回调函数的方法
```python
def _check_callback(self, callback, method):
	if (coroutines.iscoroutine(callback) or
			coroutines.iscoroutinefunction(callback)):
		raise TypeError(
			f"coroutines cannot be used with {method}()")
	if not callable(callback):
		raise TypeError(
			f'a callable object was expected by {method}(), '
			f'got {callback!r}')
```
### def _call_soon
```python
def _call_soon(self, callback, args, context):
	# 创建一个回调对象
	handle = events.Handle(callback, args, self, context)
	if handle._source_traceback:
		del handle._source_traceback[-1]
	# 添加到准备好调用的队列中
	self._ready.append(handle)
	return handle
```
### def _check_thread
检查线程是否是运行事件循环的线程
```python
def _check_thread(self):
	if self._thread_id is None:
		return
	thread_id = threading.get_ident()
	if thread_id != self._thread_id:
		raise RuntimeError(
			"Non-thread-safe operation invoked on an event loop other "
			"than the current one")
```
### def call_soon_threadsafe
线程安全的调用函数
```python
def call_soon_threadsafe(self, callback, *args, context=None):
	self._check_closed()
	if self._debug:
		self._check_callback(callback, 'call_soon_threadsafe')
	handle = self._call_soon(callback, args, context)
	if handle._source_traceback:
		del handle._source_traceback[-1]
	self._write_to_self()
	return handle
```
### def run_in_executor
在调度器中执行函数，默认是线程池调度器。
```python
def run_in_executor(self, executor, func, *args):
	self._check_closed()
	if self._debug:
		self._check_callback(func, 'run_in_executor')
	if executor is None:
		executor = self._default_executor
		if executor is None:
			executor = concurrent.futures.ThreadPoolExecutor()
			self._default_executor = executor
	return futures.wrap_future(
		executor.submit(func, *args), loop=self)
```
### def set_default_executor
设置默认的调度器
```python
def set_default_executor(self, executor):
	self._default_executor = executor
```
### def _getaddrinfo_debug
调试模式下的`addrinfo`
```python
def _getaddrinfo_debug(self, host, port, family, type, proto, flags):
	msg = [f"{host}:{port!r}"]
	if family:
		msg.append(f'family={family!r}')
	if type:
		msg.append(f'type={type!r}')
	if proto:
		msg.append(f'proto={proto!r}')
	if flags:
		msg.append(f'flags={flags!r}')
	msg = ', '.join(msg)
	logger.debug('Get address info %s', msg)

	t0 = self.time()
	addrinfo = socket.getaddrinfo(host, port, family, type, proto, flags)
	dt = self.time() - t0

	msg = f'Getting address info {msg} took {dt * 1e3:.3f}ms: {addrinfo!r}'
	if dt >= self.slow_callback_duration:
		logger.info(msg)
	else:
		logger.debug(msg)
	return addrinfo
```
### async def getaddrinfo
```python
async def getaddrinfo(self, host, port, *,
					  family=0, type=0, proto=0, flags=0):
	if self._debug:
		getaddr_func = self._getaddrinfo_debug
	else:
		getaddr_func = socket.getaddrinfo

	return await self.run_in_executor(
		None, getaddr_func, host, port, family, type, proto, flags)
```
### async def getnameinfo
获取套接字名称信息。
```python
async def getnameinfo(self, sockaddr, flags=0):
	return await self.run_in_executor(
		None, socket.getnameinfo, sockaddr, flags)
```
### async def sock_sendfile
通过sock发送文件
```python
async def sock_sendfile(self, sock, file, offset=0, count=None,
						*, fallback=True):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	#检查文件和套接字是否符合发送要求
	self._check_sendfile_params(sock, file, offset, count)
	try:
		return await self._sock_sendfile_native(sock, file,
												offset, count)
	except events.SendfileNotAvailableError as exc:
		if not fallback:
			raise
	return await self._sock_sendfile_fallback(sock, file,
											  offset, count)
```
### async def _sock_sendfile_native
```python
async def _sock_sendfile_native(self, sock, file, offset, count):
	# NB: sendfile syscall is not supported for SSL sockets and
	# non-mmap files even if sendfile is supported by OS
	raise events.SendfileNotAvailableError(
		f"syscall sendfile is not available for socket {sock!r} "
		"and file {file!r} combination")
```
### async def _sock_sendfile_fallback
```python
async def _sock_sendfile_fallback(self, sock, file, offset, count):
	if offset:
		file.seek(offset)
	blocksize = (
		min(count, constants.SENDFILE_FALLBACK_READBUFFER_SIZE)
		if count else constants.SENDFILE_FALLBACK_READBUFFER_SIZE
	)
	buf = bytearray(blocksize)
	total_sent = 0
	try:
		while True:
			if count:
				blocksize = min(count - total_sent, blocksize)
				if blocksize <= 0:
					break
			view = memoryview(buf)[:blocksize]
			read = await self.run_in_executor(None, file.readinto, view)
			if not read:
				break  # EOF
			await self.sock_sendall(sock, view[:read])
			total_sent += read
		return total_sent
	finally:
		if total_sent > 0 and hasattr(file, 'seek'):
			file.seek(offset + total_sent)
```
### def _check_sendfile_params
检查发送文件的参数，是不是符合要求。
```python
def _check_sendfile_params(self, sock, file, offset, count):
	if 'b' not in getattr(file, 'mode', 'b'):
		raise ValueError("file should be opened in binary mode")
	if not sock.type == socket.SOCK_STREAM:
		raise ValueError("only SOCK_STREAM type sockets are supported")
	if count is not None:
		if not isinstance(count, int):
			raise TypeError(
				"count must be a positive integer (got {!r})".format(count))
		if count <= 0:
			raise ValueError(
				"count must be a positive integer (got {!r})".format(count))
	if not isinstance(offset, int):
		raise TypeError(
			"offset must be a non-negative integer (got {!r})".format(
				offset))
	if offset < 0:
		raise ValueError(
			"offset must be a non-negative integer (got {!r})".format(
				offset))
```
### async def create_connection
创建一个连接，返回传输和协议
```python
async def create_connection(
		self, protocol_factory, host=None, port=None,
		*, ssl=None, family=0,
		proto=0, flags=0, sock=None,
		local_addr=None, server_hostname=None,
		ssl_handshake_timeout=None):
	if server_hostname is not None and not ssl:
		raise ValueError('server_hostname is only meaningful with ssl')

	# 没有主机名使用主机地址代替
	if server_hostname is None and ssl:
		if not host:
			raise ValueError('You must set server_hostname '
							 'when using ssl without a host')
		server_hostname = host

	if ssl_handshake_timeout is not None and not ssl:
		raise ValueError(
			'ssl_handshake_timeout is only meaningful with ssl')

	if host is not None or port is not None:
		if sock is not None:
			raise ValueError(
				'host/port and sock can not be specified at the same time')

		infos = await self._ensure_resolved(
			(host, port), family=family,
			type=socket.SOCK_STREAM, proto=proto, flags=flags, loop=self)
		if not infos:
			raise OSError('getaddrinfo() returned empty list')

		if local_addr is not None:
			laddr_infos = await self._ensure_resolved(
				local_addr, family=family,
				type=socket.SOCK_STREAM, proto=proto,
				flags=flags, loop=self)
			if not laddr_infos:
				raise OSError('getaddrinfo() returned empty list')

		exceptions = []
		for family, type, proto, cname, address in infos:
			try:
				sock = socket.socket(family=family, type=type, proto=proto)
				sock.setblocking(False)
				if local_addr is not None:
					for _, _, _, _, laddr in laddr_infos:
						try:
							sock.bind(laddr)
							break
						except OSError as exc:
							msg = (
								f'error while attempting to bind on '
								f'address {laddr!r}: '
								f'{exc.strerror.lower()}'
							)
							exc = OSError(exc.errno, msg)
							exceptions.append(exc)
					else:
						sock.close()
						sock = None
						continue
				if self._debug:
					logger.debug("connect %r to %r", sock, address)
				await self.sock_connect(sock, address)
			except OSError as exc:
				if sock is not None:
					sock.close()
				exceptions.append(exc)
			except:
				if sock is not None:
					sock.close()
				raise
			else:
				break
		else:
			if len(exceptions) == 1:
				raise exceptions[0]
			else:
				# If they all have the same str(), raise one.
				model = str(exceptions[0])
				if all(str(exc) == model for exc in exceptions):
					raise exceptions[0]
				# Raise a combined exception so the user can see all
				# the various error messages.
				raise OSError('Multiple exceptions: {}'.format(
					', '.join(str(exc) for exc in exceptions)))

	else:
		if sock is None:
			raise ValueError(
				'host and port was not specified and no sock specified')
		if sock.type != socket.SOCK_STREAM:
			# We allow AF_INET, AF_INET6, AF_UNIX as long as they
			# are SOCK_STREAM.
			# We support passing AF_UNIX sockets even though we have
			# a dedicated API for that: create_unix_connection.
			# Disallowing AF_UNIX in this method, breaks backwards
			# compatibility.
			raise ValueError(
				f'A Stream Socket was expected, got {sock!r}')

	transport, protocol = await self._create_connection_transport(
		sock, protocol_factory, ssl, server_hostname,
		ssl_handshake_timeout=ssl_handshake_timeout)
	if self._debug:
		# Get the socket from the transport because SSL transport closes
		# the old socket and creates a new SSL socket
		sock = transport.get_extra_info('socket')
		logger.debug("%r connected to %s:%r: (%r, %r)",
					 sock, host, port, transport, protocol)
	return transport, protocol
```
### async def _create_connection_transport
根据sock和协议工厂创建连接的传输对象
```python
async def _create_connection_transport(
		self, sock, protocol_factory, ssl,
		server_hostname, server_side=False,
		ssl_handshake_timeout=None):

	sock.setblocking(False)

	protocol = protocol_factory()
	waiter = self.create_future()
	# 根据参数类型创建不同的传输对象
	if ssl:
		sslcontext = None if isinstance(ssl, bool) else ssl
		transport = self._make_ssl_transport(
			sock, protocol, sslcontext, waiter,
			server_side=server_side, server_hostname=server_hostname,
			ssl_handshake_timeout=ssl_handshake_timeout)
	else:
		transport = self._make_socket_transport(sock, protocol, waiter)

	try:
		await waiter
	except:
		transport.close()
		raise

	return transport, protocol
```
### async def sendfile
```python
async def sendfile(self, transport, file, offset=0, count=None,
				   *, fallback=True):
	“”“
	向传输发送文件，必须是二进制模式打开的文件对象，offset标识开始的位置，count不是发送文件的大小，而是一次发送多少字节
	”“”
	if transport.is_closing():
		raise RuntimeError("Transport is closing")
	# 根据模式判断传输对象是否支持文件传输
	mode = getattr(transport, '_sendfile_compatible',
				   constants._SendfileMode.UNSUPPORTED)
	if mode is constants._SendfileMode.UNSUPPORTED:
		raise RuntimeError(
			f"sendfile is not supported for transport {transport!r}")
	if mode is constants._SendfileMode.TRY_NATIVE:
		#尝试通过本地发送文件
		try:
			return await self._sendfile_native(transport, file,
											   offset, count)
		except events.SendfileNotAvailableError as exc:
			if not fallback:
				raise

	if not fallback:
		raise RuntimeError(
			f"fallback is disabled and native sendfile is not "
			f"supported for transport {transport!r}")
	#另一种发送文件的方式
	return await self._sendfile_fallback(transport, file,
										 offset, count)
```
### async def _sendfile_native
```python
async def _sendfile_native(self, transp, file, offset, count):
	raise events.SendfileNotAvailableError(
		"sendfile syscall is not supported")
```
### async def _sendfile_fallback
```python
async def _sendfile_fallback(self, transp, file, offset, count):
	if offset:
		file.seek(offset)
	blocksize = min(count, 16384) if count else 16384
	buf = bytearray(blocksize)
	total_sent = 0
	# 发送文件的协议
	proto = _SendfileFallbackProtocol(transp)
	try:
		while True:
			if count:
				blocksize = min(count - total_sent, blocksize)
				if blocksize <= 0:
					return total_sent
			view = memoryview(buf)[:blocksize]
			read = await self.run_in_executor(None, file.readinto, view)
			if not read:
				return total_sent  # EOF
			await proto.drain()
			transp.write(view[:read])
			total_sent += read
	finally:
		if total_sent > 0 and hasattr(file, 'seek'):
			file.seek(offset + total_sent)
		await proto.restore()
```
### async def start_tls
一个使用tls的传输
```python
async def start_tls(self, transport, protocol, sslcontext, *,
					server_side=False,
					server_hostname=None,
					ssl_handshake_timeout=None):
	"""Upgrade transport to TLS.

	Return a new transport that *protocol* should start using
	immediately.
	"""
	# 检查是否支持ssl
	if ssl is None:
		raise RuntimeError('Python ssl module is not available')

	# 检查ssl上下文
	if not isinstance(sslcontext, ssl.SSLContext):
		raise TypeError(
			f'sslcontext is expected to be an instance of ssl.SSLContext, '
			f'got {sslcontext!r}')

	# 检查传输是否被tls支持
	if not getattr(transport, '_start_tls_compatible', False):
		raise TypeError(
			f'transport {transport!r} is not supported by start_tls()')

	waiter = self.create_future()
	# 创建一个ssl协议对象
	ssl_protocol = sslproto.SSLProtocol(
		self, protocol, sslcontext, waiter,
		server_side, server_hostname,
		ssl_handshake_timeout=ssl_handshake_timeout,
		call_connection_made=False)

	# Pause early so that "ssl_protocol.data_received()" doesn't
	# have a chance to get called before "ssl_protocol.connection_made()".
	transport.pause_reading()
	# 给传输设置一个新的协议
	transport.set_protocol(ssl_protocol)
	# 初始化
	conmade_cb = self.call_soon(ssl_protocol.connection_made, transport)
	# 注册传输套接字可读
	resume_cb = self.call_soon(transport.resume_reading)

	try:
		await waiter
	except Exception:
		transport.close()
		conmade_cb.cancel()
		resume_cb.cancel()
		raise

	return ssl_protocol._app_transport
```
### async def create_datagram_endpoint
创建一个报文连接
```python
async def create_datagram_endpoint(self, protocol_factory,
								   local_addr=None, remote_addr=None, *,
								   family=0, proto=0, flags=0,
								   reuse_address=None, reuse_port=None,
								   allow_broadcast=None, sock=None):
	"""Create datagram connection."""
	if sock is not None:
		if sock.type != socket.SOCK_DGRAM:
			raise ValueError(
				f'A UDP Socket was expected, got {sock!r}')
		if (local_addr or remote_addr or
				family or proto or flags or
				reuse_address or reuse_port or allow_broadcast):
			# show the problematic kwargs in exception msg
			opts = dict(local_addr=local_addr, remote_addr=remote_addr,
						family=family, proto=proto, flags=flags,
						reuse_address=reuse_address, reuse_port=reuse_port,
						allow_broadcast=allow_broadcast)
			problems = ', '.join(f'{k}={v}' for k, v in opts.items() if v)
			raise ValueError(
				f'socket modifier keyword arguments can not be used '
				f'when sock is specified. ({problems})')
		sock.setblocking(False)
		r_addr = None
	else:
		if not (local_addr or remote_addr):
			if family == 0:
				raise ValueError('unexpected address family')
			addr_pairs_info = (((family, proto), (None, None)),)
		elif hasattr(socket, 'AF_UNIX') and family == socket.AF_UNIX:
			for addr in (local_addr, remote_addr):
				if addr is not None and not isinstance(addr, str):
					raise TypeError('string is expected')
			addr_pairs_info = (((family, proto),
								(local_addr, remote_addr)), )
		else:
			# join address by (family, protocol)
			addr_infos = collections.OrderedDict()
			for idx, addr in ((0, local_addr), (1, remote_addr)):
				if addr is not None:
					assert isinstance(addr, tuple) and len(addr) == 2, (
						'2-tuple is expected')

					infos = await self._ensure_resolved(
						addr, family=family, type=socket.SOCK_DGRAM,
						proto=proto, flags=flags, loop=self)
					if not infos:
						raise OSError('getaddrinfo() returned empty list')

					for fam, _, pro, _, address in infos:
						key = (fam, pro)
						if key not in addr_infos:
							addr_infos[key] = [None, None]
						addr_infos[key][idx] = address

			# each addr has to have info for each (family, proto) pair
			addr_pairs_info = [
				(key, addr_pair) for key, addr_pair in addr_infos.items()
				if not ((local_addr and addr_pair[0] is None) or
						(remote_addr and addr_pair[1] is None))]

			if not addr_pairs_info:
				raise ValueError('can not get address information')

		exceptions = []

		if reuse_address is None:
			reuse_address = os.name == 'posix' and sys.platform != 'cygwin'

		for ((family, proto),
			 (local_address, remote_address)) in addr_pairs_info:
			sock = None
			r_addr = None
			try:
				sock = socket.socket(
					family=family, type=socket.SOCK_DGRAM, proto=proto)
				if reuse_address:
					sock.setsockopt(
						socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
				if reuse_port:
					_set_reuseport(sock)
				if allow_broadcast:
					sock.setsockopt(
						socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
				sock.setblocking(False)

				if local_addr:
					sock.bind(local_address)
				if remote_addr:
					if not allow_broadcast:
						await self.sock_connect(sock, remote_address)
					r_addr = remote_address
			except OSError as exc:
				if sock is not None:
					sock.close()
				exceptions.append(exc)
			except:
				if sock is not None:
					sock.close()
				raise
			else:
				break
		else:
			raise exceptions[0]

	protocol = protocol_factory()
	waiter = self.create_future()
	# 创建报文传输
	transport = self._make_datagram_transport(
		sock, protocol, r_addr, waiter)
	if self._debug:
		if local_addr:
			logger.info("Datagram endpoint local_addr=%r remote_addr=%r "
						"created: (%r, %r)",
						local_addr, remote_addr, transport, protocol)
		else:
			logger.debug("Datagram endpoint remote_addr=%r created: "
						 "(%r, %r)",
						 remote_addr, transport, protocol)

	try:
		await waiter
	except:
		transport.close()
		raise

	return transport, protocol
```
### async def _ensure_resolved
```python
async def _ensure_resolved(self, address, *,
						   family=0, type=socket.SOCK_STREAM,
						   proto=0, flags=0, loop):
	host, port = address[:2]
	info = _ipaddr_info(host, port, family, type, proto, *address[2:])
	if info is not None:
		# "host" is already a resolved IP.
		return [info]
	else:
		return await loop.getaddrinfo(host, port, family=family, type=type,
									  proto=proto, flags=flags)
```
### async def _create_server_getaddrinfo

```python
async def _create_server_getaddrinfo(self, host, port, family, flags):
	infos = await self._ensure_resolved((host, port), family=family,
										type=socket.SOCK_STREAM,
										flags=flags, loop=self)
	if not infos:
		raise OSError(f'getaddrinfo({host!r}) returned empty list')
	return infos
```
### async def create_server
创建一个TCP服务器
```python
async def create_server(
		self, protocol_factory, host=None, port=None,
		*,
		family=socket.AF_UNSPEC,
		flags=socket.AI_PASSIVE,
		sock=None,
		backlog=100,
		ssl=None,
		reuse_address=None,
		reuse_port=None,
		ssl_handshake_timeout=None,
		start_serving=True):
	"""Create a TCP server.

	The host parameter can be a string, in that case the TCP server is
	bound to host and port.

	The host parameter can also be a sequence of strings and in that case
	the TCP server is bound to all hosts of the sequence. If a host
	appears multiple times (possibly indirectly e.g. when hostnames
	resolve to the same IP address), the server is only bound once to that
	host.

	Return a Server object which can be used to stop the service.

	This method is a coroutine.
	"""
	if isinstance(ssl, bool):
		raise TypeError('ssl argument must be an SSLContext or None')

	if ssl_handshake_timeout is not None and ssl is None:
		raise ValueError(
			'ssl_handshake_timeout is only meaningful with ssl')

	if host is not None or port is not None:
		if sock is not None:
			raise ValueError(
				'host/port and sock can not be specified at the same time')

		if reuse_address is None:
			reuse_address = os.name == 'posix' and sys.platform != 'cygwin'
		sockets = []
		if host == '':
			hosts = [None]
		elif (isinstance(host, str) or
			  not isinstance(host, collections.abc.Iterable)):
			hosts = [host]
		else:
			hosts = host

		fs = [self._create_server_getaddrinfo(host, port, family=family,
											  flags=flags)
			  for host in hosts]
		infos = await tasks.gather(*fs, loop=self)
		infos = set(itertools.chain.from_iterable(infos))

		completed = False
		try:
			for res in infos:
				af, socktype, proto, canonname, sa = res
				try:
					sock = socket.socket(af, socktype, proto)
				except socket.error:
					# Assume it's a bad family/type/protocol combination.
					if self._debug:
						logger.warning('create_server() failed to create '
									   'socket.socket(%r, %r, %r)',
									   af, socktype, proto, exc_info=True)
					continue
				sockets.append(sock)
				if reuse_address:
					sock.setsockopt(
						socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
				if reuse_port:
					_set_reuseport(sock)
				# Disable IPv4/IPv6 dual stack support (enabled by
				# default on Linux) which makes a single socket
				# listen on both address families.
				if (_HAS_IPv6 and
						af == socket.AF_INET6 and
						hasattr(socket, 'IPPROTO_IPV6')):
					sock.setsockopt(socket.IPPROTO_IPV6,
									socket.IPV6_V6ONLY,
									True)
				try:
					sock.bind(sa)
				except OSError as err:
					raise OSError(err.errno, 'error while attempting '
								  'to bind on address %r: %s'
								  % (sa, err.strerror.lower())) from None
			completed = True
		finally:
			if not completed:
				for sock in sockets:
					sock.close()
	else:
		if sock is None:
			raise ValueError('Neither host/port nor sock were specified')
		if sock.type != socket.SOCK_STREAM:
			raise ValueError(f'A Stream Socket was expected, got {sock!r}')
		sockets = [sock]

	for sock in sockets:
		sock.setblocking(False)

	server = Server(self, sockets, protocol_factory,
					ssl, backlog, ssl_handshake_timeout)
	if start_serving:
		server._start_serving()
		# Skip one loop iteration so that all 'loop.add_reader'
		# go through.
		await tasks.sleep(0, loop=self)

	if self._debug:
		logger.info("%r is serving", server)
	return server
```
### async def connect_accepted_socket
处理一个被接受的连接
```python
async def connect_accepted_socket(
		self, protocol_factory, sock,
		*, ssl=None,
		ssl_handshake_timeout=None):
	"""Handle an accepted connection.

	This is used by servers that accept connections outside of
	asyncio but that use asyncio to handle connections.

	This method is a coroutine.  When completed, the coroutine
	returns a (transport, protocol) pair.
	"""
	if sock.type != socket.SOCK_STREAM:
		raise ValueError(f'A Stream Socket was expected, got {sock!r}')

	if ssl_handshake_timeout is not None and not ssl:
		raise ValueError(
			'ssl_handshake_timeout is only meaningful with ssl')

	transport, protocol = await self._create_connection_transport(
		sock, protocol_factory, ssl, '', server_side=True,
		ssl_handshake_timeout=ssl_handshake_timeout)
	if self._debug:
		# Get the socket from the transport because SSL transport closes
		# the old socket and creates a new SSL socket
		sock = transport.get_extra_info('socket')
		logger.debug("%r handled: (%r, %r)", sock, transport, protocol)
	return transport, protocol
```
### async def connect_read_pipe
连接到一个读取的管道，返回传输和协议
```python
async def connect_read_pipe(self, protocol_factory, pipe):
	protocol = protocol_factory()
	waiter = self.create_future()
	transport = self._make_read_pipe_transport(pipe, protocol, waiter)

	try:
		await waiter
	except:
		transport.close()
		raise

	if self._debug:
		logger.debug('Read pipe %r connected: (%r, %r)',
					 pipe.fileno(), transport, protocol)
	return transport, protocol
```
### async def connect_write_pipe
连接到一个写管道，返回传输和协议
```python
async def connect_write_pipe(self, protocol_factory, pipe):
	protocol = protocol_factory()
	waiter = self.create_future()
	transport = self._make_write_pipe_transport(pipe, protocol, waiter)

	try:
		await waiter
	except:
		transport.close()
		raise

	if self._debug:
		logger.debug('Write pipe %r connected: (%r, %r)',
					 pipe.fileno(), transport, protocol)
	return transport, protocol
```
### def _log_subprocess
```python
def _log_subprocess(self, msg, stdin, stdout, stderr):
	info = [msg]
	if stdin is not None:
		info.append(f'stdin={_format_pipe(stdin)}')
	if stdout is not None and stderr == subprocess.STDOUT:
		info.append(f'stdout=stderr={_format_pipe(stdout)}')
	else:
		if stdout is not None:
			info.append(f'stdout={_format_pipe(stdout)}')
		if stderr is not None:
			info.append(f'stderr={_format_pipe(stderr)}')
	logger.debug(' '.join(info))
```
### async def subprocess_shell
```python
async def subprocess_shell(self, protocol_factory, cmd, *,
						   stdin=subprocess.PIPE,
						   stdout=subprocess.PIPE,
						   stderr=subprocess.PIPE,
						   universal_newlines=False,
						   shell=True, bufsize=0,
						   **kwargs):
	if not isinstance(cmd, (bytes, str)):
		raise ValueError("cmd must be a string")
	if universal_newlines:
		raise ValueError("universal_newlines must be False")
	if not shell:
		raise ValueError("shell must be True")
	if bufsize != 0:
		raise ValueError("bufsize must be 0")
	protocol = protocol_factory()
	debug_log = None
	if self._debug:
		# don't log parameters: they may contain sensitive information
		# (password) and may be too long
		debug_log = 'run shell command %r' % cmd
		self._log_subprocess(debug_log, stdin, stdout, stderr)
	transport = await self._make_subprocess_transport(
		protocol, cmd, True, stdin, stdout, stderr, bufsize, **kwargs)
	if self._debug and debug_log is not None:
		logger.info('%s: %r', debug_log, transport)
	return transport, protocol
```
### async def subprocess_exec
```python
async def subprocess_exec(self, protocol_factory, program, *args,
						  stdin=subprocess.PIPE, stdout=subprocess.PIPE,
						  stderr=subprocess.PIPE, universal_newlines=False,
						  shell=False, bufsize=0, **kwargs):
	if universal_newlines:
		raise ValueError("universal_newlines must be False")
	if shell:
		raise ValueError("shell must be False")
	if bufsize != 0:
		raise ValueError("bufsize must be 0")
	popen_args = (program,) + args
	for arg in popen_args:
		if not isinstance(arg, (str, bytes)):
			raise TypeError(
				f"program arguments must be a bytes or text string, "
				f"not {type(arg).__name__}")
	protocol = protocol_factory()
	debug_log = None
	if self._debug:
		# don't log parameters: they may contain sensitive information
		# (password) and may be too long
		debug_log = f'execute program {program!r}'
		self._log_subprocess(debug_log, stdin, stdout, stderr)
	transport = await self._make_subprocess_transport(
		protocol, popen_args, False, stdin, stdout, stderr,
		bufsize, **kwargs)
	if self._debug and debug_log is not None:
		logger.info('%s: %r', debug_log, transport)
	return transport, protocol
```
### def get_exception_handler
返回异常处理程序
```python
def get_exception_handler(self):
	return self._exception_handler
```
### def set_exception_handler
设置异常处理程序
```python
def set_exception_handler(self, handler):
	if handler is not None and not callable(handler):
		raise TypeError(f'A callable object or None is expected, '
						f'got {handler!r}')
	self._exception_handler = handler
```
### def default_exception_handler
默认的异常处理程序
```python
def default_exception_handler(self, context):
	message = context.get('message')
	if not message:
		message = 'Unhandled exception in event loop'

	exception = context.get('exception')
	if exception is not None:
		exc_info = (type(exception), exception, exception.__traceback__)
	else:
		exc_info = False

	if ('source_traceback' not in context and
			self._current_handle is not None and
			self._current_handle._source_traceback):
		context['handle_traceback'] = \
			self._current_handle._source_traceback

	log_lines = [message]
	for key in sorted(context):
		if key in {'message', 'exception'}:
			continue
		value = context[key]
		if key == 'source_traceback':
			tb = ''.join(traceback.format_list(value))
			value = 'Object created at (most recent call last):\n'
			value += tb.rstrip()
		elif key == 'handle_traceback':
			tb = ''.join(traceback.format_list(value))
			value = 'Handle created at (most recent call last):\n'
			value += tb.rstrip()
		else:
			value = repr(value)
		log_lines.append(f'{key}: {value}')

	logger.error('\n'.join(log_lines), exc_info=exc_info)
```
### def call_exception_handler
调用异常调度器
```python
def call_exception_handler(self, context):
	"""Call the current event loop's exception handler.

	The context argument is a dict containing the following keys:

	- 'message': Error message;
	- 'exception' (optional): Exception object;
	- 'future' (optional): Future instance;
	- 'task' (optional): Task instance;
	- 'handle' (optional): Handle instance;
	- 'protocol' (optional): Protocol instance;
	- 'transport' (optional): Transport instance;
	- 'socket' (optional): Socket instance;
	- 'asyncgen' (optional): Asynchronous generator that caused
							 the exception.

	New keys maybe introduced in the future.

	Note: do not overload this method in an event loop subclass.
	For custom exception handling, use the
	`set_exception_handler()` method.
	"""
	if self._exception_handler is None:
		try:
			self.default_exception_handler(context)
		except Exception:
			# Second protection layer for unexpected errors
			# in the default implementation, as well as for subclassed
			# event loops with overloaded "default_exception_handler".
			logger.error('Exception in default exception handler',
						 exc_info=True)
	else:
		try:
			self._exception_handler(self, context)
		except Exception as exc:
			# Exception in the user set custom exception handler.
			try:
				# Let's try default handler.
				self.default_exception_handler({
					'message': 'Unhandled error in exception handler',
					'exception': exc,
					'context': context,
				})
			except Exception:
				# Guard 'default_exception_handler' in case it is
				# overloaded.
				logger.error('Exception in default exception handler '
							 'while handling an unexpected error '
							 'in custom exception handler',
							 exc_info=True)
```
### def _add_callback
添加一个handle对象到ready队列
```python
def _add_callback(self, handle):
	"""Add a Handle to _scheduled (TimerHandle) or _ready."""
	assert isinstance(handle, events.Handle), 'A Handle is required here'
	if handle._cancelled:
		return
	assert not isinstance(handle, events.TimerHandle)
	self._ready.append(handle)
```
### def _add_callback_signalsafe
从信号处理器调用的
```python
def _add_callback_signalsafe(self, handle):
	"""Like _add_callback() but called from a signal handler."""
	self._add_callback(handle)
	self._write_to_self()
```
### def _timer_handle_cancelled
定时的handle被取消时调用
```python
def _timer_handle_cancelled(self, handle):
	"""Notification that a TimerHandle has been cancelled."""
	if handle._scheduled:
		self._timer_cancelled_count += 1
```
### def _run_once
运行一次完整的事件循环
```python
def _run_once(self):
	# 定时调用handle的数量
	sched_count = len(self._scheduled)
	# 数量大于100并且取消调用的数量超过一半，跟新列表删除所有取消的调度
	if (sched_count > _MIN_SCHEDULED_TIMER_HANDLES and
		self._timer_cancelled_count / sched_count >
			_MIN_CANCELLED_TIMER_HANDLES_FRACTION):
		# 新的定时调用handle列表
		new_scheduled = []
		for handle in self._scheduled:
			# 如果已经被取消，设置调度状态为False
			if handle._cancelled:
				handle._scheduled = False
			# 没有取消的加入新的列表
			else:
				new_scheduled.append(handle)
		# 按照调用时间排序
		heapq.heapify(new_scheduled)
		self._scheduled = new_scheduled
		self._timer_cancelled_count = 0
	# 从前向后删除直到遇到第一个未取消的调度
	else:
		# Remove delayed calls that were cancelled from head of queue.
		while self._scheduled and self._scheduled[0]._cancelled:
			self._timer_cancelled_count -= 1
			handle = heapq.heappop(self._scheduled)
			handle._scheduled = False

	timeout = None
	# 如果有即时执行的调度，超时设置为0
	if self._ready or self._stopping:
		timeout = 0
	# 如果全部都是延时执行的调度，超时设置为最近的时间
	elif self._scheduled:
		# Compute the desired timeout.
		when = self._scheduled[0]._when
		timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

	# 查看是否有注册的事件可用，超时设置为 timeout
	if self._debug and timeout != 0:
		t0 = self.time()
		event_list = self._selector.select(timeout)
		dt = self.time() - t0
		if dt >= 1.0:
			level = logging.INFO
		else:
			level = logging.DEBUG
		nevent = len(event_list)
		if timeout is None:
			logger.log(level, 'poll took %.3f ms: %s events',
					   dt * 1e3, nevent)
		elif nevent:
			logger.log(level,
					   'poll %.3f ms took %.3f ms: %s events',
					   timeout * 1e3, dt * 1e3, nevent)
		elif dt >= 1.0:
			logger.log(level,
					   'poll %.3f ms took %.3f ms: timeout',
					   timeout * 1e3, dt * 1e3)
	else:
		event_list = self._selector.select(timeout)
	# 执行注册事件的回调函数
	self._process_events(event_list)

	# Handle 'later' callbacks that are ready.
	end_time = self.time() + self._clock_resolution
	# 把已经准备好的定时执行的handle添加到ready队列
	while self._scheduled:
		handle = self._scheduled[0]
		if handle._when >= end_time:
			break
		handle = heapq.heappop(self._scheduled)
		handle._scheduled = False
		self._ready.append(handle)

	# 等待执行调度的数量
	ntodo = len(self._ready)
	for i in range(ntodo):
		handle = self._ready.popleft()
		if handle._cancelled:
			continue
		if self._debug:
			try:
				self._current_handle = handle
				t0 = self.time()
				handle._run()
				dt = self.time() - t0
				if dt >= self.slow_callback_duration:
					logger.warning('Executing %s took %.3f seconds',
								   _format_handle(handle), dt)
			finally:
				self._current_handle = None
		else:
			handle._run()
	handle = None  # Needed to break cycles when an exception occurs.
```
### def _set_coroutine_origin_tracking
设置堆栈条目数
```python
def _set_coroutine_origin_tracking(self, enabled):
	if bool(enabled) == bool(self._coroutine_origin_tracking_enabled):
		return
	# 开启debug模式
	if enabled:
		# 当前堆栈数量
		self._coroutine_origin_tracking_saved_depth = (
			sys.get_coroutine_origin_tracking_depth())
		# 设置系统堆栈数量
		sys.set_coroutine_origin_tracking_depth(
			constants.DEBUG_STACK_DEPTH)
	# 关闭调试模式
	else:
		# 设置系统原先的堆栈数量
		sys.set_coroutine_origin_tracking_depth(
			self._coroutine_origin_tracking_saved_depth)

	self._coroutine_origin_tracking_enabled = enabled
```
### def get_debug
```python
def get_debug(self):
	return self._debug
```
### def set_debug
```python
def set_debug(self, enabled):
	self._debug = enabled

	if self.is_running():
		self.call_soon_threadsafe(self._set_coroutine_origin_tracking, enabled)
```
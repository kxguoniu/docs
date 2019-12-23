[TOC]
# asyncio 之 unix_events.py
## 摘要
## class _UnixSelectorEventLoop
### 初始化
```python
class _UnixSelectorEventLoop(selector_events.BaseSelectorEventLoop):
	# 添加信号处理和unix套接字
    def __init__(self, selector=None):
        super().__init__(selector)
        self._signal_handlers = {}
```
### def close
```python
def close(self):
	super().close()
	# 如果python解释器没有关闭
	if not sys.is_finalizing():
		# 取出信号并执行
		for sig in list(self._signal_handlers):
			self.remove_signal_handler(sig)
	# 如果python解释器正在关闭
	else:
		# 如果存在信号处理器
		if self._signal_handlers:
			warnings.warn(f"Closing the loop {self!r} "
						  f"on interpreter shutdown "
						  f"stage, skipping signal handlers removal",
						  ResourceWarning,
						  source=self)
			self._signal_handlers.clear()
```
### def _process_self_data
处理自己的数据
```python
def _process_self_data(self, data):
	for signum in data:
		if not signum:
			# 忽略写入的空字节
			continue
		# 处理字节
		self._handle_signal(signum)
```
### def add_signal_handler
添加信号处理函数，UNIX only
```python
def add_signal_handler(self, sig, callback, *args):
	if (coroutines.iscoroutine(callback) or
			coroutines.iscoroutinefunction(callback)):
		raise TypeError("coroutines cannot be used "
						"with add_signal_handler()")
	# 检查信号
	self._check_signal(sig)
	self._check_closed()
	try:
		# 如果这不是主线程，将会引发ValueError
		signal.set_wakeup_fd(self._csock.fileno())
	except (ValueError, OSError) as exc:
		raise RuntimeError(str(exc))
	# 添加一个处理器
	handle = events.Handle(callback, args, self, None)
	# 关联信号与处理函数
	self._signal_handlers[sig] = handle

	try:
		# 注册一个虚拟信号处理程序，告诉python需要发送一个信号去唤醒文件描述符。
		# _process_self_data() 会接收信号并调用信号处理函数
		signal.signal(sig, _sighandler_noop)

		# Set SA_RESTART to limit EINTR occurrences.
		signal.siginterrupt(sig, False)
	except OSError as exc:
		del self._signal_handlers[sig]
		if not self._signal_handlers:
			try:
				signal.set_wakeup_fd(-1)
			except (ValueError, OSError) as nexc:
				logger.info('set_wakeup_fd(-1) failed: %s', nexc)

		if exc.errno == errno.EINVAL:
			raise RuntimeError(f'sig {sig} cannot be caught')
		else:
			raise
```
### def _handle_signal
信号处理程序
```python
def _handle_signal(self, sig):
	# 获取信号绑定的处理程序
	handle = self._signal_handlers.get(sig)
	if handle is None:
		return  # Assume it's some race condition.
	# 如果处理程序已经取消，删除对该信号的处理
	if handle._cancelled:
		self.remove_signal_handler(sig)  # Remove it properly.
	# 执行函数
	else:
		self._add_callback_signalsafe(handle)
```
### def remove_signal_handler
删除信号处理程序，UNIX only
```python
def remove_signal_handler(self, sig):
	self._check_signal(sig)
	# 删除信号字典
	try:
		del self._signal_handlers[sig]
	except KeyError:
		return False

	if sig == signal.SIGINT:
		handler = signal.default_int_handler
	else:
		handler = signal.SIG_DFL
	# 注册信号处理程序为默认的函数
	try:
		signal.signal(sig, handler)
	except OSError as exc:
		if exc.errno == errno.EINVAL:
			raise RuntimeError(f'sig {sig} cannot be caught')
		else:
			raise

	if not self._signal_handlers:
		try:
			signal.set_wakeup_fd(-1)
		except (ValueError, OSError) as exc:
			logger.info('set_wakeup_fd(-1) failed: %s', exc)

	return True
```
### def _check_signal
检查信号是否有效
```python
def _check_signal(self, sig):
	if not isinstance(sig, int):
		raise TypeError(f'sig must be an int, not {sig!r}')

	if not (1 <= sig < signal.NSIG):
		raise ValueError(f'sig {sig} out of range(1, {signal.NSIG})')
```
### def _make_read_pipe_transport
```python
def _make_read_pipe_transport(self, pipe, protocol, waiter=None,
							  extra=None):
	return _UnixReadPipeTransport(self, pipe, protocol, waiter, extra)
```
### def _make_write_pipe_transport
```python
def _make_write_pipe_transport(self, pipe, protocol, waiter=None,
							   extra=None):
	return _UnixWritePipeTransport(self, pipe, protocol, waiter, extra)
```
### def _make_subprocess_transport
```python
async def _make_subprocess_transport(self, protocol, args, shell,
									 stdin, stdout, stderr, bufsize,
									 extra=None, **kwargs):
	with events.get_child_watcher() as watcher:
		waiter = self.create_future()
		transp = _UnixSubprocessTransport(self, protocol, args, shell,
										  stdin, stdout, stderr, bufsize,
										  waiter=waiter, extra=extra,
										  **kwargs)

		watcher.add_child_handler(transp.get_pid(),
								  self._child_watcher_callback, transp)
		try:
			await waiter
		except Exception:
			transp.close()
			await transp._wait()
			raise

	return transp
```
### def _child_watcher_callback
```python
def _child_watcher_callback(self, pid, returncode, transp):
	self.call_soon_threadsafe(transp._process_exited, returncode)
```
### async def create_unix_connection
创建一个 unix 连接
```python
async def create_unix_connection(
		self, protocol_factory, path=None, *,
		ssl=None, sock=None,
		server_hostname=None,
		ssl_handshake_timeout=None):
	assert server_hostname is None or isinstance(server_hostname, str)
	if ssl:
		if server_hostname is None:
			raise ValueError(
				'you have to pass server_hostname when using ssl')
	else:
		if server_hostname is not None:
			raise ValueError('server_hostname is only meaningful with ssl')
		if ssl_handshake_timeout is not None:
			raise ValueError(
				'ssl_handshake_timeout is only meaningful with ssl')

	if path is not None:
		if sock is not None:
			raise ValueError(
				'path and sock can not be specified at the same time')
		# 创建一个套接字连接到地址
		path = os.fspath(path)
		sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM, 0)
		try:
			sock.setblocking(False)
			await self.sock_connect(sock, path)
		except:
			sock.close()
			raise

	else:
		if sock is None:
			raise ValueError('no path and sock were specified')
		if (sock.family != socket.AF_UNIX or
				sock.type != socket.SOCK_STREAM):
			raise ValueError(
				f'A UNIX Domain Stream Socket was expected, got {sock!r}')
		sock.setblocking(False)
	# 创建连接的传输和协议
	transport, protocol = await self._create_connection_transport(
		sock, protocol_factory, ssl, server_hostname,
		ssl_handshake_timeout=ssl_handshake_timeout)
	return transport, protocol
```
### async def create_unix_server
```python
async def create_unix_server(
		self, protocol_factory, path=None, *,
		sock=None, backlog=100, ssl=None,
		ssl_handshake_timeout=None,
		start_serving=True):
	if isinstance(ssl, bool):
		raise TypeError('ssl argument must be an SSLContext or None')

	if ssl_handshake_timeout is not None and not ssl:
		raise ValueError(
			'ssl_handshake_timeout is only meaningful with ssl')

	if path is not None:
		if sock is not None:
			raise ValueError(
				'path and sock can not be specified at the same time')

		path = os.fspath(path)
		sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

		# Check for abstract socket. `str` and `bytes` paths are supported.
		if path[0] not in (0, '\x00'):
			try:
				if stat.S_ISSOCK(os.stat(path).st_mode):
					os.remove(path)
			except FileNotFoundError:
				pass
			except OSError as err:
				# Directory may have permissions only to create socket.
				logger.error('Unable to check or remove stale UNIX socket '
							 '%r: %r', path, err)
		# 套接字绑定地址
		try:
			sock.bind(path)
		except OSError as exc:
			sock.close()
			if exc.errno == errno.EADDRINUSE:
				# Let's improve the error message by adding
				# with what exact address it occurs.
				msg = f'Address {path!r} is already in use'
				raise OSError(errno.EADDRINUSE, msg) from None
			else:
				raise
		except:
			sock.close()
			raise
	else:
		if sock is None:
			raise ValueError(
				'path was not specified, and no sock specified')

		if (sock.family != socket.AF_UNIX or
				sock.type != socket.SOCK_STREAM):
			raise ValueError(
				f'A UNIX Domain Stream Socket was expected, got {sock!r}')
	# 设置非阻塞
	sock.setblocking(False)
	# 创建服务器
	server = base_events.Server(self, [sock], protocol_factory,
								ssl, backlog, ssl_handshake_timeout)
	# 运行服务器
	if start_serving:
		server._start_serving()
		# Skip one loop iteration so that all 'loop.add_reader'
		# go through.
		await tasks.sleep(0, loop=self)

	return server
```
### async def _sock_sendfile_native
```python
async def _sock_sendfile_native(self, sock, file, offset, count):
	try:
		os.sendfile
	except AttributeError as exc:
		raise events.SendfileNotAvailableError(
			"os.sendfile() is not available")
	try:
		fileno = file.fileno()
	except (AttributeError, io.UnsupportedOperation) as err:
		raise events.SendfileNotAvailableError("not a regular file")
	try:
		fsize = os.fstat(fileno).st_size
	except OSError as err:
		raise events.SendfileNotAvailableError("not a regular file")
	# 文件大小
	blocksize = count if count else fsize
	if not blocksize:
		return 0  # empty file

	fut = self.create_future()
	# 发送文件
	self._sock_sendfile_native_impl(fut, None, sock, fileno,
									offset, count, blocksize, 0)
	return await fut
```
### def _sock_sendfile_native_impl
```python
def _sock_sendfile_native_impl(self, fut, registered_fd, sock, fileno,
							   offset, count, blocksize, total_sent):
	fd = sock.fileno()
	if registered_fd is not None:
		# Remove the callback early.  It should be rare that the
		# selector says the fd is ready but the call still returns
		# EAGAIN, and I am willing to take a hit in that case in
		# order to simplify the common case.
		self.remove_writer(registered_fd)
	if fut.cancelled():
		self._sock_sendfile_update_filepos(fileno, offset, total_sent)
		return
	if count:
		blocksize = count - total_sent
		if blocksize <= 0:
			self._sock_sendfile_update_filepos(fileno, offset, total_sent)
			fut.set_result(total_sent)
			return

	try:
		sent = os.sendfile(fd, fileno, offset, blocksize)
	except (BlockingIOError, InterruptedError):
		if registered_fd is None:
			self._sock_add_cancellation_callback(fut, sock)
		self.add_writer(fd, self._sock_sendfile_native_impl, fut,
						fd, sock, fileno,
						offset, count, blocksize, total_sent)
	except OSError as exc:
		if (registered_fd is not None and
				exc.errno == errno.ENOTCONN and
				type(exc) is not ConnectionError):
			# If we have an ENOTCONN and this isn't a first call to
			# sendfile(), i.e. the connection was closed in the middle
			# of the operation, normalize the error to ConnectionError
			# to make it consistent across all Posix systems.
			new_exc = ConnectionError(
				"socket is not connected", errno.ENOTCONN)
			new_exc.__cause__ = exc
			exc = new_exc
		if total_sent == 0:
			# We can get here for different reasons, the main
			# one being 'file' is not a regular mmap(2)-like
			# file, in which case we'll fall back on using
			# plain send().
			err = events.SendfileNotAvailableError(
				"os.sendfile call failed")
			self._sock_sendfile_update_filepos(fileno, offset, total_sent)
			fut.set_exception(err)
		else:
			self._sock_sendfile_update_filepos(fileno, offset, total_sent)
			fut.set_exception(exc)
	except Exception as exc:
		self._sock_sendfile_update_filepos(fileno, offset, total_sent)
		fut.set_exception(exc)
	else:
		if sent == 0:
			# EOF
			self._sock_sendfile_update_filepos(fileno, offset, total_sent)
			fut.set_result(total_sent)
		else:
			offset += sent
			total_sent += sent
			if registered_fd is None:
				self._sock_add_cancellation_callback(fut, sock)
			self.add_writer(fd, self._sock_sendfile_native_impl, fut,
							fd, sock, fileno,
							offset, count, blocksize, total_sent)
```
### def _sock_sendfile_update_filepos
```python
def _sock_sendfile_update_filepos(self, fileno, offset, total_sent):
	if total_sent > 0:
		os.lseek(fileno, offset, os.SEEK_SET)
```
### def _sock_add_cancellation_callback
```python
def _sock_add_cancellation_callback(self, fut, sock):
	def cb(fut):
		if fut.cancelled():
			fd = sock.fileno()
			if fd != -1:
				self.remove_writer(fd)
	fut.add_done_callback(cb)
```
## class _UnixReadPipeTransport
### 初始化
```python
class _UnixReadPipeTransport(transports.ReadTransport):
    max_size = 256 * 1024  # 一次事件循环中读取的最大字节数

    def __init__(self, loop, pipe, protocol, waiter=None, extra=None):
        super().__init__(extra)
		# 额外信息
        self._extra['pipe'] = pipe
        self._loop = loop
        self._pipe = pipe
		# 管道的文件描述符
        self._fileno = pipe.fileno()
        self._protocol = protocol
		# 管道的状态
        self._closing = False
		# 文件描述符类型
        mode = os.fstat(self._fileno).st_mode
		# 判断管道是不是属于特定类型
        if not (stat.S_ISFIFO(mode) or
                stat.S_ISSOCK(mode) or
                stat.S_ISCHR(mode)):
            self._pipe = None
            self._fileno = None
            self._protocol = None
            raise ValueError("Pipe transport is for pipes/sockets only.")
		# 设置管道为非阻塞
        os.set_blocking(self._fileno, False)
		# 初始化协议
        self._loop.call_soon(self._protocol.connection_made, self)
        # 注册管道可读监控事件
        self._loop.call_soon(self._loop._add_reader,
                             self._fileno, self._read_ready)
		# 如果存在等待的 future，设置future的结果
        if waiter is not None:
            # only wake up the waiter when connection_made() has been called
            self._loop.call_soon(futures._set_result_unless_cancelled,
                                 waiter, None)
```
### def _read_ready
管道的可读事件处理函数
```python
def _read_ready(self):
	# 从管道中读取数据
	try:
		data = os.read(self._fileno, self.max_size)
	except (BlockingIOError, InterruptedError):
		pass
	except OSError as exc:
		self._fatal_error(exc, 'Fatal read error on pipe transport')
	else:
		# 读取到数据，把数据写入到协议中
		if data:
			self._protocol.data_received(data)
		# 没有读取到数据，管道已经关闭，重置传输
		else:
			if self._loop.get_debug():
				logger.info("%r was closed by peer", self)
			self._closing = True
			self._loop._remove_reader(self._fileno)
			self._loop.call_soon(self._protocol.eof_received)
			self._loop.call_soon(self._call_connection_lost, None)
```
### def pause_reading
暂停读取数据
```python
def pause_reading(self):
	self._loop._remove_reader(self._fileno)
```
### def resume_reading
恢复读取数据
```python
def resume_reading(self):
	self._loop._add_reader(self._fileno, self._read_ready)
```
### def set_protocol/get_protocol
```python
def set_protocol(self, protocol):
	self._protocol = protocol

def get_protocol(self):
	return self._protocol
```
### def is_closing/close
```python
def is_closing(self):
	return self._closing

def close(self):
	if not self._closing:
		self._close(None)

def _close(self, exc):
	self._closing = True
	self._loop._remove_reader(self._fileno)
	self._loop.call_soon(self._call_connection_lost, exc)
```
### def __del__
```python
def __del__(self):
	if self._pipe is not None:
		warnings.warn(f"unclosed transport {self!r}", ResourceWarning,
					  source=self)
		self._pipe.close()
```
### def _fatal_error
```python
def _fatal_error(self, exc, message='Fatal error on pipe transport'):
	# should be called by exception handler only
	if (isinstance(exc, OSError) and exc.errno == errno.EIO):
		if self._loop.get_debug():
			logger.debug("%r: %s", self, message, exc_info=True)
	else:
		self._loop.call_exception_handler({
			'message': message,
			'exception': exc,
			'transport': self,
			'protocol': self._protocol,
		})
	self._close(exc)
```
### def _call_connection_lost
```python
def _call_connection_lost(self, exc):
	try:
		self._protocol.connection_lost(exc)
	finally:
		self._pipe.close()
		self._pipe = None
		self._protocol = None
		self._loop = None
```
## class _UnixWritePipeTransport
### 初始化
```python
class _UnixWritePipeTransport(transports._FlowControlMixin,
                              transports.WriteTransport):

    def __init__(self, loop, pipe, protocol, waiter=None, extra=None):
        super().__init__(extra, loop)
        self._extra['pipe'] = pipe
        self._pipe = pipe
        self._fileno = pipe.fileno()
        self._protocol = protocol
		# 缓冲区
        self._buffer = bytearray()
        self._conn_lost = 0
		# 写传输的状态
        self._closing = False  # Set when close() or write_eof() called.

        mode = os.fstat(self._fileno).st_mode
        is_char = stat.S_ISCHR(mode)
        is_fifo = stat.S_ISFIFO(mode)
        is_socket = stat.S_ISSOCK(mode)
		# 如果管道描述符不是特定的模式
        if not (is_char or is_fifo or is_socket):
            self._pipe = None
            self._fileno = None
            self._protocol = None
            raise ValueError("Pipe transport is only for "
                             "pipes, sockets and character devices")
		# 设置管道为非阻塞
        os.set_blocking(self._fileno, False)
		# 初始化协议
        self._loop.call_soon(self._protocol.connection_made, self)

        if is_socket or (is_fifo and not sys.platform.startswith("aix")):
            # 注册管道可读事件监控
            self._loop.call_soon(self._loop._add_reader,
                                 self._fileno, self._read_ready)
		# 如果等待的 future 不为空，设置结果
        if waiter is not None:
            # only wake up the waiter when connection_made() has been called
            self._loop.call_soon(futures._set_result_unless_cancelled,
                                 waiter, None)
```
### def get_write_buffer_size
获取写缓冲区的大小
```python
def get_write_buffer_size(self):
	return len(self._buffer)
```
### def _read_ready
管道可读事件处理函数
```python
def _read_ready(self):
	# Pipe was closed by peer.
	if self._loop.get_debug():
		logger.info("%r was closed by peer", self)
	# 如果缓冲区还有数据
	if self._buffer:
		self._close(BrokenPipeError())
	else:
		self._close()
```
### def write
使用管道写入数据
```python
def write(self, data):
	assert isinstance(data, (bytes, bytearray, memoryview)), repr(data)
	# 如果数据是字节数组，转化成缓冲区数据
	if isinstance(data, bytearray):
		data = memoryview(data)
	if not data:
		return
	# 如果连接丢失或者传输关闭
	if self._conn_lost or self._closing:
		if self._conn_lost >= constants.LOG_THRESHOLD_FOR_CONNLOST_WRITES:
			logger.warning('pipe closed by peer or '
						   'os.write(pipe, data) raised exception.')
		self._conn_lost += 1
		return
	# 如果缓冲区没有数据
	if not self._buffer:
		# 尝试先发送数据
		try:
			n = os.write(self._fileno, data)
		except (BlockingIOError, InterruptedError):
			n = 0
		except Exception as exc:
			self._conn_lost += 1
			self._fatal_error(exc, 'Fatal write error on pipe transport')
			return
		# 数据发送完毕
		if n == len(data):
			return
		# 数据发送一部分，删除已经发送的部分
		elif n > 0:
			data = memoryview(data)[n:]
		# 添加可写事件监控
		self._loop._add_writer(self._fileno, self._write_ready)
	# 把数据添加到缓冲区
	self._buffer += data
	# 检查协议是否需要暂停
	self._maybe_pause_protocol()
```
### def _write_ready
管道可写事件处理函数
```python
def _write_ready(self):
	assert self._buffer, 'Data should not be empty'
	# 发送缓冲区里面的数据
	try:
		n = os.write(self._fileno, self._buffer)
	except (BlockingIOError, InterruptedError):
		pass
	except Exception as exc:
		self._buffer.clear()
		self._conn_lost += 1
		# Remove writer here, _fatal_error() doesn't it
		# because _buffer is empty.
		self._loop._remove_writer(self._fileno)
		self._fatal_error(exc, 'Fatal write error on pipe transport')
	else:
		# 缓冲区数据发送完毕
		if n == len(self._buffer):
			# 清空缓冲区
			self._buffer.clear()
			# 删除可写事件监控
			self._loop._remove_writer(self._fileno)
			# 恢复协议向传输中写入数据
			self._maybe_resume_protocol()  # May append to buffer.
			# 如果传输已经关闭
			if self._closing:
				# 删除可读事件监控，用于关闭传输的
				self._loop._remove_reader(self._fileno)
				# 连接丢失
				self._call_connection_lost(None)
			return
		# 删除缓冲区中已经发送的数据
		elif n > 0:
			del self._buffer[:n]
```
### def write_eof
管道写完毕
```python
def can_write_eof(self):
	return True

def write_eof(self):
	if self._closing:
		return
	assert self._pipe
	# 设置管道状态为关闭
	self._closing = True
	# 如果没有缓冲区数据
	if not self._buffer:
		# 删除管道可读事件监控
		self._loop._remove_reader(self._fileno)
		# 连接丢失
		self._loop.call_soon(self._call_connection_lost, None)
```
### def set_protocol/get_protocol
```python
def set_protocol(self, protocol):
	self._protocol = protocol

def get_protocol(self):
	return self._protocol
```
### def is_closing/close
```python
def is_closing(self):
	return self._closing

def close(self):
	if self._pipe is not None and not self._closing:
		# write_eof is all what we needed to close the write pipe
		self.write_eof()

def _close(self, exc=None):
	self._closing = True
	if self._buffer:
		self._loop._remove_writer(self._fileno)
	self._buffer.clear()
	self._loop._remove_reader(self._fileno)
	self._loop.call_soon(self._call_connection_lost, exc)
```
### def __del__
```python
def __del__(self):
	if self._pipe is not None:
		warnings.warn(f"unclosed transport {self!r}", ResourceWarning,
					  source=self)
		self._pipe.close()
```
### def abort
```python
def abort(self):
	self._close(None)
```
### def _fatal_error
```
def _fatal_error(self, exc, message='Fatal error on pipe transport'):
	# should be called by exception handler only
	if isinstance(exc, OSError):
		if self._loop.get_debug():
			logger.debug("%r: %s", self, message, exc_info=True)
	else:
		self._loop.call_exception_handler({
			'message': message,
			'exception': exc,
			'transport': self,
			'protocol': self._protocol,
		})
	self._close(exc)
```
### def _call_connection_lost
```python
def _call_connection_lost(self, exc):
	try:
		self._protocol.connection_lost(exc)
	finally:
		self._pipe.close()
		self._pipe = None
		self._protocol = None
		self._loop = None
```
## class _UnixSubprocessTransport
```python
class _UnixSubprocessTransport(base_subprocess.BaseSubprocessTransport):
    def _start(self, args, shell, stdin, stdout, stderr, bufsize, **kwargs):
        stdin_w = None
        if stdin == subprocess.PIPE:
            # Use a socket pair for stdin, since not all platforms
            # support selecting read events on the write end of a
            # socket (which we use in order to detect closing of the
            # other end).  Notably this is needed on AIX, and works
            # just fine on other platforms.
            stdin, stdin_w = socket.socketpair()
        try:
            self._proc = subprocess.Popen(
                args, shell=shell, stdin=stdin, stdout=stdout, stderr=stderr,
                universal_newlines=False, bufsize=bufsize, **kwargs)
            if stdin_w is not None:
                stdin.close()
                self._proc.stdin = open(stdin_w.detach(), 'wb', buffering=bufsize)
                stdin_w = None
        finally:
            if stdin_w is not None:
                stdin.close()
                stdin_w.close()
```
## class _UnixDefaultEventLoopPolicy
带有子进程监视器的事件循环策略
### 初始化
```python
class _UnixDefaultEventLoopPolicy(events.BaseDefaultEventLoopPolicy):
    _loop_factory = _UnixSelectorEventLoop

    def __init__(self):
        super().__init__()
        self._watcher = None
```
### def _init_watcher
```python
def _init_watcher(self):
	with events._lock:
		if self._watcher is None:  # pragma: no branch
			self._watcher = SafeChildWatcher()
			if isinstance(threading.current_thread(),
						  threading._MainThread):
				self._watcher.attach_loop(self._local._loop)
```
### def set_event_loop
```python
def set_event_loop(self, loop):
	"""Set the event loop.

	As a side effect, if a child watcher was set before, then calling
	.set_event_loop() from the main thread will call .attach_loop(loop) on
	the child watcher.
	"""

	super().set_event_loop(loop)

	if (self._watcher is not None and
			isinstance(threading.current_thread(), threading._MainThread)):
		self._watcher.attach_loop(loop)
```
### def get_child_watcher
```python
def get_child_watcher(self):
	"""Get the watcher for child processes.

	If not yet set, a SafeChildWatcher object is automatically created.
	"""
	if self._watcher is None:
		self._init_watcher()

	return self._watcher
```
### def set_child_watcher
```python
def set_child_watcher(self, watcher):
	"""Set the watcher for child processes."""

	assert watcher is None or isinstance(watcher, AbstractChildWatcher)

	if self._watcher is not None:
		self._watcher.close()

	self._watcher = watcher
```
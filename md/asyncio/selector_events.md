[TOC]
# asyncio 之 select_events.py
## 摘要

## class BaseSelectorEventLoop
基础的选择器事件循环
### 初始化
```python
class BaseSelectorEventLoop(base_events.BaseEventLoop):
    def __init__(self, selector=None):
        super().__init__()
		# 如果没有选择器，则选择系统上的最优实现
        if selector is None:
            selector = selectors.DefaultSelector()
        logger.debug('Using selector: %s', selector.__class__.__name__)
        self._selector = selector
		# 创建一个管道
        self._make_self_pipe()
		# 传输的弱引用字典
        self._transports = weakref.WeakValueDictionary()
```
### def _make_socket_transport
创建一个套接字传输
```python
def _make_socket_transport(self, sock, protocol, waiter=None, *,
						   extra=None, server=None):
	return _SelectorSocketTransport(self, sock, protocol, waiter,
									extra, server)
```
### def _make_ssl_transport
创建基于 ssl 的套接字传输
```python
def _make_ssl_transport(
		self, rawsock, protocol, sslcontext, waiter=None,
		*, server_side=False, server_hostname=None,
		extra=None, server=None,
		ssl_handshake_timeout=constants.SSL_HANDSHAKE_TIMEOUT):
	ssl_protocol = sslproto.SSLProtocol(
			self, protocol, sslcontext, waiter,
			server_side, server_hostname,
			ssl_handshake_timeout=ssl_handshake_timeout)
	_SelectorSocketTransport(self, rawsock, ssl_protocol,
							 extra=extra, server=server)
	return ssl_protocol._app_transport
```
### def _make_datagram_transport
创建报文传输
```python
def _make_datagram_transport(self, sock, protocol,
							 address=None, waiter=None, extra=None):
	return _SelectorDatagramTransport(self, sock, protocol,
									  address, waiter, extra)
```
### def close
关闭事件循环
```python
def close(self):
	if self.is_running():
		raise RuntimeError("Cannot close a running event loop")
	if self.is_closed():
		return
	# 关闭管道
	self._close_self_pipe()
	super().close()
	# 关闭选择器
	if self._selector is not None:
		self._selector.close()
		self._selector = None
```
### def _close_self_pipe
关闭创建的管道
```python
def _close_self_pipe(self):
	# 删除管道的可读事件监控
	self._remove_reader(self._ssock.fileno())
	# 关闭 server socket
	self._ssock.close()
	self._ssock = None
	# 关闭 client socket
	self._csock.close()
	self._csock = None
	# 文件描述符减一
	self._internal_fds -= 1
```
### def _make_self_pipe
创建管道
```python
def _make_self_pipe(self):
	# 创建管道的两端
	self._ssock, self._csock = socket.socketpair()
	# 设置非阻塞
	self._ssock.setblocking(False)
	self._csock.setblocking(False)
	# 文件描述符增加
	self._internal_fds += 1
	# 注册可读事件监控
	self._add_reader(self._ssock.fileno(), self._read_from_self)
```
### def _process_self_data
```python
def _process_self_data(self, data):
	pass
```
### def _read_from_self
接收来自自己的数据
```python
def _read_from_self(self):
	while True:
		try:
			data = self._ssock.recv(4096)
			if not data:
				break
			self._process_self_data(data)
		except InterruptedError:
			continue
		except BlockingIOError:
			break
```
### def _write_to_self
向自己发送数据
```python
def _write_to_self(self):
	# This may be called from a different thread, possibly after
	# _close_self_pipe() has been called or even while it is
	# running.  Guard for self._csock being None or closed.  When
	# a socket is closed, send() raises OSError (with errno set to
	# EBADF, but let's not rely on the exact error code).
	csock = self._csock
	if csock is not None:
		try:
			csock.send(b'\0')
		except OSError:
			if self._debug:
				logger.debug("Fail to write a null byte into the "
							 "self-pipe socket",
							 exc_info=True)
```
### def _start_serving
启动服务器
```python
def _start_serving(self, protocol_factory, sock,
				   sslcontext=None, server=None, backlog=100,
				   ssl_handshake_timeout=constants.SSL_HANDSHAKE_TIMEOUT):
	self._add_reader(sock.fileno(), self._accept_connection,
					 protocol_factory, sock, sslcontext, server, backlog,
					 ssl_handshake_timeout)
```
## class _SelectorTransport
### 初始化
```python
class _SelectorTransport(transports._FlowControlMixin,
                         transports.Transport):
	# 接受数据的缓存大小
    max_size = 256 * 1024  # Buffer size passed to recv().
	# 缓冲区使用的函数
    _buffer_factory = bytearray  # Constructs initial value for self._buffer.

    _sock = None

    def __init__(self, loop, sock, protocol, extra=None, server=None):
        super().__init__(extra, loop)
		# 额外的套接字信息
        self._extra['socket'] = sock
        try:
            self._extra['sockname'] = sock.getsockname()
        except OSError:
            self._extra['sockname'] = None
        if 'peername' not in self._extra:
            try:
                self._extra['peername'] = sock.getpeername()
            except socket.error:
                self._extra['peername'] = None
		# 套接字对象
        self._sock = sock
		# 文件描述符
        self._sock_fd = sock.fileno()

        self._protocol_connected = False
		# 设置传输的协议
        self.set_protocol(protocol)

        self._server = server
		# 缓冲区对象
        self._buffer = self._buffer_factory()
		# 连接丢失
        self._conn_lost = 0  # Set when call to connection_lost scheduled.
        # 传输关闭状态
		self._closing = False  # Set when close() called.
        if self._server is not None:
            self._server._attach()
		# 在 loop 中注册自己
        loop._transports[self._sock_fd] = self
```
### def abort
```python
def abort(self):
	self._force_close(None)
```
### def set_protocol
设置传输的协议以及协议标识符
```python
def set_protocol(self, protocol):
	self._protocol = protocol
	self._protocol_connected = True
```
### def get_protocol
```python
def get_protocol(self):
	return self._protocol
```
### def is_closing
```python
def is_closing(self):
	return self._closing
```
### def close
关闭传输
```python
def close(self):
	if self._closing:
		return
	self._closing = True
	# 从loop中删除自己
	self._loop._remove_reader(self._sock_fd)
	# 如果缓冲区没有数据
	if not self._buffer:
		# 连接丢失增加一
		self._conn_lost += 1
		# 删除可写文件描述符
		self._loop._remove_writer(self._sock_fd)
		# 通过loop 执行函数
		self._loop.call_soon(self._call_connection_lost, None)
```
### def __del__
```python
def __del__(self):
	# 如果套接字还在，关闭套接字
	if self._sock is not None:
		warnings.warn(f"unclosed transport {self!r}", ResourceWarning,
					  source=self)
		self._sock.close()
```
### def _fatal_error
```python
def _fatal_error(self, exc, message='Fatal error on transport'):
	# 只能从异常处理器调用
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
	self._force_close(exc)
```
### def _force_close
强制关闭
```python
def _force_close(self, exc):
	if self._conn_lost:
		return
	# 如果存在缓冲区，清空并取消注册的可写文件描述符
	if self._buffer:
		self._buffer.clear()
		self._loop._remove_writer(self._sock_fd)
	# 如果传输还没有关闭，设置状态为关闭，删除可读文件描述符
	if not self._closing:
		self._closing = True
		self._loop._remove_reader(self._sock_fd)
	# 连接丢失增加一
	self._conn_lost += 1
	# 尽快执行
	self._loop.call_soon(self._call_connection_lost, exc)
```
### def _call_connection_lost
```python
def _call_connection_lost(self, exc):
	# 如果协议连接中，关闭协议的连接
	try:
		if self._protocol_connected:
			self._protocol.connection_lost(exc)
	finally:
		# 关闭套接字
		self._sock.close()
		self._sock = None
		self._protocol = None
		self._loop = None
		server = self._server
		# 关闭server
		if server is not None:
			server._detach()
			self._server = None
```
### def get_write_buffer_size
```python
def get_write_buffer_size(self):
	return len(self._buffer)
```
### def _add_reader
```python
def _add_reader(self, fd, callback, *args):
	if self._closing:
		return
	# 添加可读调用
	self._loop._add_reader(fd, callback, *args)
```
## class _SelectorSocketTransport
### 初始化
```python
class _SelectorSocketTransport(_SelectorTransport):

    _start_tls_compatible = True
    _sendfile_compatible = constants._SendfileMode.TRY_NATIVE

    def __init__(self, loop, sock, protocol, waiter=None,
                 extra=None, server=None):
		# 可读回调对象
        self._read_ready_cb = None
        super().__init__(loop, sock, protocol, extra, server)
		# 结束标识符
        self._eof = False
		# 暂停状态
        self._paused = False
		#
        self._empty_waiter = None

        # Disable the Nagle algorithm -- small writes will be
        # sent without waiting for the TCP ACK.  This generally
        # decreases the latency (in some cases significantly.)
        base_events._set_nodelay(self._sock)

		# 连接协议
        self._loop.call_soon(self._protocol.connection_made, self)
		# 添加可读事件
        self._loop.call_soon(self._add_reader,
                             self._sock_fd, self._read_ready)
        if waiter is not None:
            # only wake up the waiter when connection_made() has been called
            self._loop.call_soon(futures._set_result_unless_cancelled,
                                 waiter, None)
```
### def set_protocol
设置协议
```python
def set_protocol(self, protocol):
	# 如果是缓冲区协议的实例设置可读回调
	if isinstance(protocol, protocols.BufferedProtocol):
		self._read_ready_cb = self._read_ready__get_buffer
	else:
		self._read_ready_cb = self._read_ready__data_received

	super().set_protocol(protocol)
```
### def is_reading
```python
def is_reading(self):
	return not self._paused and not self._closing
```
### def pause_reading
暂停读取数据
```python
def pause_reading(self):
	if self._closing or self._paused:
		return
	# 设置暂停状态
	self._paused = True
	# 删除注册的可读文件描述符
	self._loop._remove_reader(self._sock_fd)
	if self._loop.get_debug():
		logger.debug("%r pauses reading", self)
```
### def resume_reading
恢复可读状态
```python
def resume_reading(self):
	if self._closing or not self._paused:
		return
	# 取消暂停
	self._paused = False
	# 添加可读事件监控
	self._add_reader(self._sock_fd, self._read_ready)
	if self._loop.get_debug():
		logger.debug("%r resumes reading", self)
```
### def _read_ready
可读事件调用的函数
```python
def _read_ready(self):
	self._read_ready_cb()
```
### def _read_ready__get_buffer
缓冲区协议的可读回调函数
```python
def _read_ready__get_buffer(self):
	if self._conn_lost:
		return
	# 获取一个不限制区大小的缓冲区
	try:
		buf = self._protocol.get_buffer(-1)
		if not len(buf):
			raise RuntimeError('get_buffer() returned an empty buffer')
	except Exception as exc:
		self._fatal_error(
			exc, 'Fatal error: protocol.get_buffer() call failed.')
		return

	# 从套接字中接受数据保存到缓冲区
	try:
		nbytes = self._sock.recv_into(buf)
	except (BlockingIOError, InterruptedError):
		return
	except Exception as exc:
		self._fatal_error(exc, 'Fatal read error on socket transport')
		return

	# 如果没有读取到数据说明已经结束
	if not nbytes:
		self._read_ready__on_eof()
		return
	# 更新缓冲区的数据
	try:
		self._protocol.buffer_updated(nbytes)
	except Exception as exc:
		self._fatal_error(
			exc, 'Fatal error: protocol.buffer_updated() call failed.')
```
### def _read_ready__data_received
非缓冲区协议可读回调函数
```python
def _read_ready__data_received(self):
	if self._conn_lost:
		return
	# 从套接字中读取数据
	try:
		data = self._sock.recv(self.max_size)
	except (BlockingIOError, InterruptedError):
		return
	except Exception as exc:
		self._fatal_error(exc, 'Fatal read error on socket transport')
		return
	# 如果没有读取到数据说明已经结束
	if not data:
		self._read_ready__on_eof()
		return
	# 把数据发送到协议
	try:
		self._protocol.data_received(data)
	except Exception as exc:
		self._fatal_error(
			exc, 'Fatal error: protocol.data_received() call failed.')
```
### def _read_ready__on_eof
数据读取完毕
```python
def _read_ready__on_eof(self):
	if self._loop.get_debug():
		logger.debug("%r received EOF", self)
	# 调用协议的读取结束方法
	try:
		keep_open = self._protocol.eof_received()
	except Exception as exc:
		self._fatal_error(
			exc, 'Fatal error: protocol.eof_received() call failed.')
		return

	# 保持连接打开，但是只能写入不能读取数据
	if keep_open:
		self._loop._remove_reader(self._sock_fd)
	else:
		self.close()
```
### def write
写入数据
```python
def write(self, data):
	# 数据必须是字节数据或者缓存数据
	if not isinstance(data, (bytes, bytearray, memoryview)):
		raise TypeError(f'data argument must be a bytes-like object, '
						f'not {type(data).__name__!r}')
	# 如果写已经结束
	if self._eof:
		raise RuntimeError('Cannot call write() after write_eof()')
	# 如果有文件正在写入
	if self._empty_waiter is not None:
		raise RuntimeError('unable to write; sendfile is in progress')
	if not data:
		return

	if self._conn_lost:
		if self._conn_lost >= constants.LOG_THRESHOLD_FOR_CONNLOST_WRITES:
			logger.warning('socket.send() raised exception.')
		self._conn_lost += 1
		return
	# 如果传输的缓冲区没有数据
	if not self._buffer:
		# Optimization: try to send now.
		# 尝试发送数据，返回发送数据的长度
		try:
			n = self._sock.send(data)
		except (BlockingIOError, InterruptedError):
			pass
		except Exception as exc:
			self._fatal_error(exc, 'Fatal write error on socket transport')
			return
		# 去掉已经发送的数据
		else:
			data = data[n:]
			if not data:
				return
		# Not all was written; register write handler.
		self._loop._add_writer(self._sock_fd, self._write_ready)

	# 把要发送的数据添加到缓冲区
	self._buffer.extend(data)
	# 数据可能超过缓冲区最大限制，暂停协议写入数据
	self._maybe_pause_protocol()
```
### def _write_ready
```python
def _write_ready(self):
	assert self._buffer, 'Data should not be empty'

	if self._conn_lost:
		return
	# 发送缓冲区的数据，返回发送的长度
	try:
		n = self._sock.send(self._buffer)
	except (BlockingIOError, InterruptedError):
		pass
	except Exception as exc:
		self._loop._remove_writer(self._sock_fd)
		self._buffer.clear()
		self._fatal_error(exc, 'Fatal write error on socket transport')
		if self._empty_waiter is not None:
			self._empty_waiter.set_exception(exc)
	else:
		# 从缓冲区删除发送的数据
		if n:
			del self._buffer[:n]
		# 缓冲区数据可能小于最低限制，开启协议写入数据
		self._maybe_resume_protocol()  # May append to buffer.
		# 如果缓冲区数据为空
		if not self._buffer:
			# 删除可写事件文件描述符
			self._loop._remove_writer(self._sock_fd)
			# 如果是写入文件，设置结果
			if self._empty_waiter is not None:
				self._empty_waiter.set_result(None)
			# 如果传输关闭
			if self._closing:
				self._call_connection_lost(None)
			# 如果写入结束，关闭套接字连接
			elif self._eof:
				self._sock.shutdown(socket.SHUT_WR)
```
### def write_eof
```python
def write_eof(self):
	if self._closing or self._eof:
		return
	# 设置写结束标识符
	self._eof = True
	# 如果没有缓冲区数据，关闭写连接。否则等缓冲区数据发送完毕再写入。
	if not self._buffer:
		self._sock.shutdown(socket.SHUT_WR)
```
### def can_write_eof
```python
def can_write_eof(self):
	return True
```
### def _call_connection_lost
```python
def _call_connection_lost(self, exc):
	super()._call_connection_lost(exc)
	# 文件发送不为空
	if self._empty_waiter is not None:
		self._empty_waiter.set_exception(
			ConnectionError("Connection is closed by peer"))
```
### def _make_empty_waiter
```python
def _make_empty_waiter(self):
	if self._empty_waiter is not None:
		raise RuntimeError("Empty waiter is already set")
	# 创建一个future对象
	self._empty_waiter = self._loop.create_future()
	# 如果缓冲区数据为空，设置结果
	if not self._buffer:
		self._empty_waiter.set_result(None)
	return self._empty_waiter
```
### def _reset_empty_waiter
```python
def _reset_empty_waiter(self):
	self._empty_waiter = None
```
## class _SelectorDatagramTransport
```python
class _SelectorDatagramTransport(_SelectorTransport):
	# 数据缓冲在队列中
    _buffer_factory = collections.deque

    def __init__(self, loop, sock, protocol, address=None,
                 waiter=None, extra=None):
        super().__init__(loop, sock, protocol, extra)
		# 地址
        self._address = address
		# 连接协议
        self._loop.call_soon(self._protocol.connection_made, self)
		# 添加可读事件回调
        self._loop.call_soon(self._add_reader,
                             self._sock_fd, self._read_ready)
        if waiter is not None:
            # only wake up the waiter when connection_made() has been called
			self._loop.call_soon(futures._set_result_unless_cancelled,
                                 waiter, None)
```
### def get_write_buffer_size
获取写缓冲区大小
```python
def get_write_buffer_size(self):
	return sum(len(data) for data, _ in self._buffer)
```
### def _read_ready
可读回调方法
```python
def _read_ready(self):
	if self._conn_lost:
		return
	# 从套接字读取数据
	try:
		data, addr = self._sock.recvfrom(self.max_size)
	except (BlockingIOError, InterruptedError):
		pass
	except OSError as exc:
		self._protocol.error_received(exc)
	except Exception as exc:
		self._fatal_error(exc, 'Fatal read error on datagram transport')
	# 把接收到的数据发送给协议
	else:
		self._protocol.datagram_received(data, addr)
```
### def sendto
发送数据
```python
def sendto(self, data, addr=None):
	if not isinstance(data, (bytes, bytearray, memoryview)):
		raise TypeError(f'data argument must be a bytes-like object, '
						f'not {type(data).__name__!r}')
	if not data:
		return
	# 如果存在地址
	if self._address:
		if addr not in (None, self._address):
			raise ValueError(
				f'Invalid address: must be None or {self._address}')
		addr = self._address

	if self._conn_lost and self._address:
		if self._conn_lost >= constants.LOG_THRESHOLD_FOR_CONNLOST_WRITES:
			logger.warning('socket.send() raised exception.')
		self._conn_lost += 1
		return
	# 缓冲区没有数据
	if not self._buffer:
		#尝试发送数据
		try:
			if self._extra['peername']:
				self._sock.send(data)
			else:
				self._sock.sendto(data, addr)
			return
		except (BlockingIOError, InterruptedError):
			self._loop._add_writer(self._sock_fd, self._sendto_ready)
		except OSError as exc:
			self._protocol.error_received(exc)
			return
		except Exception as exc:
			self._fatal_error(
				exc, 'Fatal write error on datagram transport')
			return

	# 确保缓冲区是不可变的
	self._buffer.append((bytes(data), addr))
	self._maybe_pause_protocol()
```
### def _sendto_ready
发送数据的方法
```python
def _sendto_ready(self):
	# 如果缓冲区存在数据
	while self._buffer:
		# 取出缓冲区的数据
		data, addr = self._buffer.popleft()
		try:
			if self._extra['peername']:
				self._sock.send(data)
			else:
				self._sock.sendto(data, addr)
		except (BlockingIOError, InterruptedError):
			self._buffer.appendleft((data, addr))  # Try again later.
			break
		except OSError as exc:
			self._protocol.error_received(exc)
			return
		except Exception as exc:
			self._fatal_error(
				exc, 'Fatal write error on datagram transport')
			return

	self._maybe_resume_protocol()  # May append to buffer.
	# 缓冲区没有数据了
	if not self._buffer:
		# 删除可写事件
		self._loop._remove_writer(self._sock_fd)
		# 如果传输关闭了，关闭连接
		if self._closing:
			self._call_connection_lost(None)
```
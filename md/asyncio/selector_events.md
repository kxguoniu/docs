[TOC]
## 摘要
作为承上(base_events)启下(unix_events or windows_events)的文件，实现了基础选择器事件循环类(类中实现的方法大多都是与传输和管道相关的)，和套接字传输类以及报文传输类的实现。
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
创建内部使用的管道
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
处理信号的方法，由子类重写。
```python
def _process_self_data(self, data):
	pass
```
### def _read_from_self
接收管道发过来的信号并调用对应的handle
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
通过管道接收的信号已经被处理，发送通知。
```python
def _write_to_self(self):
	# 可能在不同的线程中调用，可能是正在关闭过程中调用，如果发送数据的时候连接已经关闭，会引发异常。
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
由`Server`对象调用，开始监听套接字并处理连接的请求。
```python
def _start_serving(self, protocol_factory, sock,
				   sslcontext=None, server=None, backlog=100,
				   ssl_handshake_timeout=constants.SSL_HANDSHAKE_TIMEOUT):
	self._add_reader(sock.fileno(), self._accept_connection,
					 protocol_factory, sock, sslcontext, server, backlog,
					 ssl_handshake_timeout)
```
### def _accept_connection
接收一个请求并处理。
```python
def _accept_connection(
		self, protocol_factory, sock,
		sslcontext=None, server=None, backlog=100,
		ssl_handshake_timeout=constants.SSL_HANDSHAKE_TIMEOUT):
	# 当套接字读取事件被触发，此函数只调用一次，可能有多个连接在等待，所以在循环中处理他们。
	# 一次处理限定数量的连接
	for _ in range(backlog):
		# 接收一个连接
		try:
			conn, addr = sock.accept()
			if self._debug:
				logger.debug("%r got a new connection from %r: %r",
							 server, addr, conn)
			conn.setblocking(False)
		except (BlockingIOError, InterruptedError, ConnectionAbortedError):
			return None
		except OSError as exc:
			# There's nowhere to send the error, so just log it.
			if exc.errno in (errno.EMFILE, errno.ENFILE,
							 errno.ENOBUFS, errno.ENOMEM):
				# Some platforms (e.g. Linux keep reporting the FD as
				# ready, so we remove the read handler temporarily.
				# We'll try again in a while.
				self.call_exception_handler({
					'message': 'socket.accept() out of system resource',
					'exception': exc,
					'socket': sock,
				})
				# 删除套接字的读监听事件
				self._remove_reader(sock.fileno())
				# 一秒之后重新注册套接字的读取
				self.call_later(constants.ACCEPT_RETRY_DELAY,
								self._start_serving,
								protocol_factory, sock, sslcontext, server,
								backlog, ssl_handshake_timeout)
			else:
				raise  # The event loop will catch, log and ignore it.
		else:
			extra = {'peername': addr}
			# 创建一个处理单个连接的协程
			accept = self._accept_connection2(
				protocol_factory, conn, extra, sslcontext, server,
				ssl_handshake_timeout)
			# 创建一个task执行协程
			self.create_task(accept)
```
### async def _accept_connection2
具体处理服务器接受连接的方法
```python
async def _accept_connection2(
		self, protocol_factory, conn, extra,
		sslcontext=None, server=None,
		ssl_handshake_timeout=constants.SSL_HANDSHAKE_TIMEOUT):
	protocol = None
	transport = None
	try:
		# 创建连接协议
		protocol = protocol_factory()
		# 创建一个 future
		waiter = self.create_future()
		# 如果使用 ssl，创建 ssl 传输
		if sslcontext:
			transport = self._make_ssl_transport(
				conn, protocol, sslcontext, waiter=waiter,
				server_side=True, extra=extra, server=server,
				ssl_handshake_timeout=ssl_handshake_timeout)
		# 否则创建一般的传输
		else:
			transport = self._make_socket_transport(
				conn, protocol, waiter=waiter, extra=extra,
				server=server)
		# 等待创建传输结束
		try:
			await waiter
		except:
			transport.close()
			raise

		# 现在由协议来处理连接，除了异常
		# 协议会调用回调函数去处理读取流和写入流
	except Exception as exc:
		if self._debug:
			context = {
				'message':
					'Error on transport creation for incoming connection',
				'exception': exc,
			}
			if protocol is not None:
				context['protocol'] = protocol
			if transport is not None:
				context['transport'] = transport
			self.call_exception_handler(context)
```
### def _ensure_fd_no_transport
确定文件描述符没有被传输占用。
```python
def _ensure_fd_no_transport(self, fd):
	fileno = fd
	if not isinstance(fileno, int):
		try:
			fileno = int(fileno.fileno())
		except (AttributeError, TypeError, ValueError):
			# This code matches selectors._fileobj_to_fd function.
			raise ValueError(f"Invalid file object: {fd!r}") from None
	try:
		transport = self._transports[fileno]
	except KeyError:
		pass
	else:
		if not transport.is_closing():
			raise RuntimeError(
				f'File descriptor {fd!r} is used by transport '
				f'{transport!r}')
```
### def _add_reader
注册文件描述符的可读事件监控
```python
def _add_reader(self, fd, callback, *args):
	self._check_closed()
	# 创建一个回调函数处理器
	handle = events.Handle(callback, args, self, None)
	# 尝试从选择器中获取文件描述符注册的数据
	try:
		key = self._selector.get_key(fd)
	# 如果文件描述符没有被注册，直接注册
	except KeyError:
		self._selector.register(fd, selectors.EVENT_READ,
								(handle, None))
	# 文件描述符已经被注册
	else:
		mask, (reader, writer) = key.events, key.data
		# 重新注册文件描述符，更新注册的附加数据
		self._selector.modify(fd, mask | selectors.EVENT_READ,
							  (handle, writer))
		# 取消之前注册的可读事件回调
		if reader is not None:
			reader.cancel()
```
### def _remove_reader
删除文件描述符的可读事件监控
```python
def _remove_reader(self, fd):
	if self.is_closed():
		return False
	# 获取注册的数据
	try:
		key = self._selector.get_key(fd)
	except KeyError:
		return False
	else:
		mask, (reader, writer) = key.events, key.data
		mask &= ~selectors.EVENT_READ
		# 如果只注册了可读事件监控，直接取消文件描述符的监控
		if not mask:
			self._selector.unregister(fd)
		# 如果还注册了读写监控，重新注册成为可写监控
		else:
			self._selector.modify(fd, mask, (None, writer))
		# 取消可读监控回调函数
		if reader is not None:
			reader.cancel()
			return True
		else:
			return False
```
### def _add_writer
添加套接字可写事件监控
```python
def _add_writer(self, fd, callback, *args):
	self._check_closed()
	# 创建一个handle
	handle = events.Handle(callback, args, self, None)
	# 尝试获取该文件描述符已经注册的信息
	try:
		key = self._selector.get_key(fd)
	# 没有被注册过，直接注册
	except KeyError:
		self._selector.register(fd, selectors.EVENT_WRITE,
								(None, handle))
	else:
		# 取出之前注册的信息
		mask, (reader, writer) = key.events, key.data
		# 重新注册
		self._selector.modify(fd, mask | selectors.EVENT_WRITE,
							  (reader, handle))
		# 取消之前注册的回调函数
		if writer is not None:
			writer.cancel()
```
### def _remove_writer
删除套接字的可写事件监控
```python
def _remove_writer(self, fd):
	"""Remove a writer callback."""
	if self.is_closed():
		return False
	# 取出注册的事件信息
	try:
		key = self._selector.get_key(fd)
	except KeyError:
		return False
	else:
		mask, (reader, writer) = key.events, key.data
		# Remove both writer and connector.
		mask &= ~selectors.EVENT_WRITE
		# 如果只注册了可写事件监控，直接取消注册
		if not mask:
			self._selector.unregister(fd)
		# 如果注册了读写事件监控，重新注册，去掉可写事件监控
		else:
			self._selector.modify(fd, mask, (reader, None))
		# 取消可写事件的回调函数
		if writer is not None:
			writer.cancel()
			return True
		else:
			return False
```
### def add_reader
```python
def add_reader(self, fd, callback, *args):
	self._ensure_fd_no_transport(fd)
	return self._add_reader(fd, callback, *args)
```
### def remove_reader
```python
def remove_reader(self, fd):
	self._ensure_fd_no_transport(fd)
	return self._remove_reader(fd)
```
### def add_writer
```python
def add_writer(self, fd, callback, *args):
	self._ensure_fd_no_transport(fd)
	return self._add_writer(fd, callback, *args)
```
### def remove_writer
```python
def remove_writer(self, fd):
	self._ensure_fd_no_transport(fd)
	return self._remove_writer(fd)
```
### async def sock_recv
从套接字接收数据并返回
```python
async def sock_recv(self, sock, n):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	fut = self.create_future()
	# 接收数据
	self._sock_recv(fut, None, sock, n)
	# 等待接收数据完成返回
	return await fut
```
### def _sock_recv
```python
def _sock_recv(self, fut, registered_fd, sock, n):
	# 如果不能立即执行，会把自己添加为 I/O 回调
	if registered_fd is not None:
		# 删除可读事件监控
		self.remove_reader(registered_fd)
	if fut.cancelled():
		return
	# 接收数据
	try:
		data = sock.recv(n)
	except (BlockingIOError, InterruptedError):
		# 出现异常，添加可读事件监控
		fd = sock.fileno()
		self.add_reader(fd, self._sock_recv, fut, fd, sock, n)
	except Exception as exc:
		fut.set_exception(exc)
	# 接收完毕，设置结果
	else:
		fut.set_result(data)
```
### async def sock_recv_into
接收到的数据被存在缓冲区，返回接收的大小
```python
async def sock_recv_into(self, sock, buf):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	fut = self.create_future()
	self._sock_recv_into(fut, None, sock, buf)
	return await fut
```
### def _sock_recv_into
参考 `_sock_recv_into`
```python
def _sock_recv_into(self, fut, registered_fd, sock, buf):
	if registered_fd is not None:
		self.remove_reader(registered_fd)
	if fut.cancelled():
		return
	try:
		nbytes = sock.recv_into(buf)
	except (BlockingIOError, InterruptedError):
		fd = sock.fileno()
		self.add_reader(fd, self._sock_recv_into, fut, fd, sock, buf)
	except Exception as exc:
		fut.set_exception(exc)
	else:
		fut.set_result(nbytes)
```
### async def sock_sendall
向套接字发送数据，如果出现异常，无法确定接收端处理了多少
```python
async def sock_sendall(self, sock, data):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	fut = self.create_future()
	if data:
		self._sock_sendall(fut, None, sock, data)
	else:
		fut.set_result(None)
	return await fut
```
### def _sock_sendall
```python
def _sock_sendall(self, fut, registered_fd, sock, data):
	if registered_fd is not None:
		self.remove_writer(registered_fd)
	if fut.cancelled():
		return

	try:
		n = sock.send(data)
	except (BlockingIOError, InterruptedError):
		n = 0
	except Exception as exc:
		fut.set_exception(exc)
		return
	# 如果发送完成，返回
	if n == len(data):
		fut.set_result(None)
	# 否则继续发送接下来的数据
	else:
		if n:
			data = data[n:]
		fd = sock.fileno()
		self.add_writer(fd, self._sock_sendall, fut, fd, sock, data)
```
### async def sock_connect
根据地址连接到远程套接字
```python
async def sock_connect(self, sock, address):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")

	if not hasattr(socket, 'AF_UNIX') or sock.family != socket.AF_UNIX:
		# 如果不是 unix 套接字家族
		resolved = await self._ensure_resolved(
			address, family=sock.family, proto=sock.proto, loop=self)
		_, _, _, _, address = resolved[0]

	fut = self.create_future()
	# 连接套接字
	self._sock_connect(fut, sock, address)
	return await fut
```
### def _sock_connect
```python
def _sock_connect(self, fut, sock, address):
	# 取出套接字的文件描述符
	fd = sock.fileno()
	# 尝试连接地址
	try:
		sock.connect(address)
	except (BlockingIOError, InterruptedError):
		# 当C函数连接失败，连接在后台运行。我们必须等待直到连接成功或者失败时通知套接字可写
		# 连接完成后删除可写时间监控
		fut.add_done_callback(
			functools.partial(self._sock_connect_done, fd))
		# 添加可写事件监控，监控连接事件
		self.add_writer(fd, self._sock_connect_cb, fut, sock, address)
	except Exception as exc:
		fut.set_exception(exc)
	# 连接成功
	else:
		fut.set_result(None)
```
### def _sock_connect_done
```python
def _sock_connect_done(self, fd, fut):
	self.remove_writer(fd)
```
### def _sock_connect_cb
```python
def _sock_connect_cb(self, fut, sock, address):
	if fut.cancelled():
		return

	try:
		err = sock.getsockopt(socket.SOL_SOCKET, socket.SO_ERROR)
		if err != 0:
			# Jump to any except clause below.
			raise OSError(err, f'Connect call failed {address}')
	except (BlockingIOError, InterruptedError):
		# 套接字仍然注册，回调将稍后重试
		pass
	except Exception as exc:
		fut.set_exception(exc)
	else:
		fut.set_result(None)
```
### async def sock_accept
接受一个请求，返回套接字和地址
```python
async def sock_accept(self, sock):
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	fut = self.create_future()
	self._sock_accept(fut, False, sock)
	return await fut
```
### def _sock_accept
```python
def _sock_accept(self, fut, registered, sock):
	fd = sock.fileno()
	# 如果套接字已经注册，删除可读事件监控
	if registered:
		self.remove_reader(fd)
	if fut.cancelled():
		return
	# 套接字接受一个连接，并设置非阻塞
	try:
		conn, address = sock.accept()
		conn.setblocking(False)
	except (BlockingIOError, InterruptedError):
		# 连接出现异常，添加可读监控，监听其他进入的连接
		self.add_reader(fd, self._sock_accept, fut, True, sock)
	except Exception as exc:
		fut.set_exception(exc)
	else:
		fut.set_result((conn, address))
```
### async def _sendfile_native
跳过传输直接使用套接字发送文件，被父类调用。
```python
async def _sendfile_native(self, transp, file, offset, count):
	# 删除传输的文件描述符
	del self._transports[transp._sock_fd]
	# 如果传输没有处于暂停状态
	resume_reading = transp.is_reading()
	# 传输暂停读取数据
	transp.pause_reading()
	# 等待传输发送缓冲区为空
	await transp._make_empty_waiter()
	# 尝试发送文件
	try:
		return await self.sock_sendfile(transp._sock, file, offset, count,
										fallback=False)
	finally:
		# 传输取消占用
		transp._reset_empty_waiter()
		# 如果可以，恢复读取数据
		if resume_reading:
			transp.resume_reading()
		# 注册传输
		self._transports[transp._sock_fd] = transp
```
### def _process_events
处理注册的文件描述符的事件回调函数，被`loop.run_once`调用.
```python
def _process_events(self, event_list):
	for key, mask in event_list:
		fileobj, (reader, writer) = key.fileobj, key.data
		# 如果是注册的可读事件，删除注册，调用回调函数。
		if mask & selectors.EVENT_READ and reader is not None:
			if reader._cancelled:
				self._remove_reader(fileobj)
			else:
				self._add_callback(reader)
		# 如果注册的可写事件，删除注册，调用回调函数。
		if mask & selectors.EVENT_WRITE and writer is not None:
			if writer._cancelled:
				self._remove_writer(fileobj)
			else:
				self._add_callback(writer)
```
### def _stop_serving
停止 Server 服务
```python
def _stop_serving(self, sock):
	# 删除可读事件监控，不在接受请求。关闭套接字。
	self._remove_reader(sock.fileno())
	sock.close()
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

		# 服务器对象，标识是不是服务器的传输对象
        self._server = server
		# 缓冲区对象
        self._buffer = self._buffer_factory()
		# 连接丢失
        self._conn_lost = 0  # Set when call to connection_lost scheduled.
        # 传输关闭状态
		self._closing = False  # Set when close() called.
		# 如果是服务器的传输对象，连接数增加一
        if self._server is not None:
            self._server._attach()
		# 在 loop 中注册自己
        loop._transports[self._sock_fd] = self
```
### def abort
强制关闭
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
致命的错误，强制关闭传输。
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
		# 如果是服务器的传输对象，服务器连接对象减一
		if server is not None:
			server._detach()
			self._server = None
```
### def get_write_buffer_size
获取传输写缓冲区当前的大小
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
		# 传输套接字被占用发送文件
        self._empty_waiter = None

        # Disable the Nagle algorithm -- small writes will be
        # sent without waiting for the TCP ACK.  This generally
        # decreases the latency (in some cases significantly.)
        base_events._set_nodelay(self._sock)

		# 连接协议初始化
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
设置协议，根据协议类型选择接收套接字数据的方法
```python
def set_protocol(self, protocol):
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
暂停传输从套接字读取数据
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
恢复传输从套接字读取数据
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
套接字数据可读回调的方法
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
	# 把数据通过协议发送到读取流
	try:
		self._protocol.data_received(data)
	except Exception as exc:
		self._fatal_error(
			exc, 'Fatal error: protocol.data_received() call failed.')
```
### def _read_ready__on_eof
套接字数据读取完毕
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
传输向套接字发送数据
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
	# 检查是否需要暂停协议，阻止写入流向传输发送数据
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
		# 检查传输是否可以接收数据
		self._maybe_resume_protocol()  # May append to buffer.
		# 如果缓冲区数据为空
		if not self._buffer:
			# 删除可写事件文件描述符
			self._loop._remove_writer(self._sock_fd)
			# 唤醒需要使用套接字发送文件的协程
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
	# 如果没有缓冲区数据，写入完毕。否则等缓冲区数据发送完毕再写入。
	if not self._buffer:
		self._sock.shutdown(socket.SHUT_WR)
```
### def can_write_eof
```python
def can_write_eof(self):
	return True
```
### def _call_connection_lost
连接丢失回调函数
```python
def _call_connection_lost(self, exc):
	super()._call_connection_lost(exc)
	# 如果传输套接字正在发送文件
	if self._empty_waiter is not None:
		self._empty_waiter.set_exception(
			ConnectionError("Connection is closed by peer"))
```
### def _make_empty_waiter
等待传输写缓冲区为空就暂停传输向套接字发送数据，因为文件传输要占用。
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
文件发送完毕，取消限制
```python
def _reset_empty_waiter(self):
	self._empty_waiter = None
```
## class _SelectorDatagramTransport
### 初始化
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
套接字可读回调函数
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
	# 把接受到的数据通过协议发送给读取流
	else:
		self._protocol.datagram_received(data, addr)
```
### def sendto
报文传输发送数据
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
发送缓冲区中的数据
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
[TOC]
# asyncio 之 sslproto.py
## 摘要

## def _create_transport_context
创建传输的ssl上下文，只有客户端的传输才可以使用
```python
def _create_transport_context(server_side, server_hostname):
    if server_side:
        raise ValueError('Server side SSL needs a valid SSLContext')

    sslcontext = ssl.create_default_context()
    if not server_hostname:
        sslcontext.check_hostname = False
    return sslcontext
```

## class _SSLPipe
### 初始化
```python
class _SSLPipe(object):
    max_size = 256 * 1024   # Buffer size passed to read()
    def __init__(self, context, server_side, server_hostname=None):
        # 用于SSLContext的参数
        self._context = context
        # 标识是服务器段传输还是客户端传输
        self._server_side = server_side
        # 用于指定要连接的服务器主机名
        self._server_hostname = server_hostname
        # ssl 初始状态
        self._state = _UNWRAPPED
        self._incoming = ssl.MemoryBIO()
        self._outgoing = ssl.MemoryBIO()
        # ssl对象
        self._sslobj = None
        self._need_ssldata = False
        # 握手完成回调函数
        self._handshake_cb = None
        # 连接关闭回调函数
        self._shutdown_cb = None
```
### def context
ssl 上下文
```python
@property
def context(self):
	return self._context
```
### def ssl_object
ssl 对象
```python
@property
def ssl_object(self):
	return self._sslobj
```
### def need_ssldata
连接需要的数据
```python
@property
def need_ssldata(self):
	return self._need_ssldata
```
### def wrapped
当前连接是否已经完成
```python
@property
def wrapped(self):
	return self._state == _WRAPPED
```
### def do_handshake
开始 ssl 握手， 完成时调用回调函数，参数为None或者异常
```python
def do_handshake(self, callback=None):
	# 返回一个ssldata数据列表
	if self._state != _UNWRAPPED:
		raise RuntimeError('handshake in progress or completed')
	self._sslobj = self._context.wrap_bio(
		self._incoming, self._outgoing,
		server_side=self._server_side,
		server_hostname=self._server_hostname)
	# 状态设置为正在进行握手
	self._state = _DO_HANDSHAKE
	# 设置完成回调函数
	self._handshake_cb = callback
	ssldata, appdata = self.feed_ssldata(b'', only_handshake=True)
	assert len(appdata) == 0
	return ssldata
```
### def shutdown
关闭 ssl，完成时调用回调函数
```python
def shutdown(self, callback=None):
	# 关闭ssl， 返回ssldata列表
	if self._state == _UNWRAPPED:
		raise RuntimeError('no security layer present')
	if self._state == _SHUTDOWN:
		raise RuntimeError('shutdown in progress')
	assert self._state in (_WRAPPED, _DO_HANDSHAKE)
	# 更新状态为关闭
	self._state = _SHUTDOWN
	# 设置关闭回调函数
	self._shutdown_cb = callback
	# 发送断开连接信息
	ssldata, appdata = self.feed_ssldata(b'')
	assert appdata == [] or appdata == [b'']
	return ssldata
```
### def feed_eof
发送结束
```python
def feed_eof(self):
	self._incoming.write_eof()
	ssldata, appdata = self.feed_ssldata(b'')
	assert appdata == [] or appdata == [b'']
```
### def feed_ssldata
通过ssl发送数据
```python
def feed_ssldata(self, data, only_handshake=False):
	# 将 ssl 记录级数据输入到管道中，数据是bytes实例
	# 返回一个(ssldata,appdata)元组， ssldata是一个发送到远程ssl的数据的缓冲区列表
	# 如果没有握手，直接返回
	if self._state == _UNWRAPPED:
		if data:
			appdata = [data]
		else:
			appdata = []
		return ([], appdata)

	self._need_ssldata = False
	# 如果存在数据，加密应用发送的数据
	if data:
		self._incoming.write(data)

	ssldata = []
	appdata = []
	try:
		# 如果ssl正在握手
		if self._state == _DO_HANDSHAKE:
			# 执行握手，直到握手完成
			self._sslobj.do_handshake()
			# 设置状态为握手完成
			self._state = _WRAPPED
			# 如果存在回调函数，调用
			if self._handshake_cb:
				self._handshake_cb(None)
			# 如果仅仅是握手的操作，直接返回。
			if only_handshake:
				return (ssldata, appdata)
		# 如果 ssl已经建立连接
		if self._state == _WRAPPED:
			# 从ssl中读取数据添加到应用数据列表
			while True:
				chunk = self._sslobj.read(self.max_size)
				appdata.append(chunk)
				if not chunk:  # close_notify
					break
		# 如果ssl连接断开，关闭ssl
		elif self._state == _SHUTDOWN:
			self._sslobj.unwrap()
			self._sslobj = None
			self._state = _UNWRAPPED
			# 调用关闭回调函数
			if self._shutdown_cb:
				self._shutdown_cb()
		# 如果状态是还没有开始握手
		elif self._state == _UNWRAPPED:
			appdata.append(self._incoming.read())
	except (ssl.SSLError, ssl.CertificateError) as exc:
		exc_errno = getattr(exc, 'errno', None)
		if exc_errno not in (
				ssl.SSL_ERROR_WANT_READ, ssl.SSL_ERROR_WANT_WRITE,
				ssl.SSL_ERROR_SYSCALL):
			# 握手出现异常，调用回调函数
			if self._state == _DO_HANDSHAKE and self._handshake_cb:
				self._handshake_cb(exc)
			raise
		self._need_ssldata = (exc_errno == ssl.SSL_ERROR_WANT_READ)

	# 检查需要发送的数据，发生在最初的握手和重新谈判的时候
	if self._outgoing.pending:
		ssldata.append(self._outgoing.read())
	return (ssldata, appdata)
```
### def feed_appdata
从ssl接收数据
```python
def feed_appdata(self, data, offset=0):
	# 将明文数据输入管道，返回 (ssldata, offset)
	assert 0 <= offset <= len(data)
	if self._state == _UNWRAPPED:
		# 以非包装的模式传送数据
		if offset < len(data):
			ssldata = [data[offset:]]
		else:
			ssldata = []
		return (ssldata, len(data))

	ssldata = []
	view = memoryview(data)
	while True:
		self._need_ssldata = False
		try:
			if offset < len(view):
				offset += self._sslobj.write(view[offset:])
		except ssl.SSLError as exc:
			exc_errno = getattr(exc, 'errno', None)
			if exc.reason == 'PROTOCOL_IS_SHUTDOWN':
				exc_errno = exc.errno = ssl.SSL_ERROR_WANT_READ
			if exc_errno not in (ssl.SSL_ERROR_WANT_READ,
								 ssl.SSL_ERROR_WANT_WRITE,
								 ssl.SSL_ERROR_SYSCALL):
				raise
			self._need_ssldata = (exc_errno == ssl.SSL_ERROR_WANT_READ)

		if self._outgoing.pending:
			ssldata.append(self._outgoing.read())
		if offset == len(view) or self._need_ssldata:
			break
	return (ssldata, offset)
```
## class _SSLProtocolTransport
### 初始化
```python
class _SSLProtocolTransport(transports._FlowControlMixin,
                            transports.Transport):

    # 发送文件的模式
    _sendfile_compatible = constants._SendfileMode.FALLBACK

    def __init__(self, loop, ssl_protocol):
        self._loop = loop
        # ssl协议实例
        self._ssl_protocol = ssl_protocol
        self._closed = False
```
### def get_extra_info
获取传输的额外信息
```python
def get_extra_info(self, name, default=None):
	return self._ssl_protocol._get_extra_info(name, default)
```
### def set_protocol
设置app侧的协议
```python
def set_protocol(self, protocol):
	self._ssl_protocol._set_app_protocol(protocol)
```
### def get_protocol
获取app侧的协议
```python
def get_protocol(self):
	return self._ssl_protocol._app_protocol
```
### def is_closing
传输是否关闭
```python
def is_closing(self):
	return self._closed
```
### def close
设置传输关闭状态，调用ssl协议的方法关闭
```python
def close(self):
	# 关闭传输，不再接受数据，现有数据将异步刷新，数据发送完毕调用协议的连接丢失
	self._closed = True
	self._ssl_protocol._start_shutdown()
```
### 调用ssl协议中传输的方法
获取传输是否正在读取数据
```python
def is_reading(self):
	tr = self._ssl_protocol._transport
	if tr is None:
		raise RuntimeError('SSL transport has not been initialized yet')
	return tr.is_reading()

def pause_reading(self):
	self._ssl_protocol._transport.pause_reading()

def resume_reading(self):
	self._ssl_protocol._transport.resume_reading()

def set_write_buffer_limits(self, high=None, low=None):
	self._ssl_protocol._transport.set_write_buffer_limits(high, low)

def get_write_buffer_size(self):
	return self._ssl_protocol._transport.get_write_buffer_size()

@property
def _protocol_paused(self):
	return self._ssl_protocol._transport._protocol_paused
```
### def write
```python
def write(self, data):
	# 向传输缓冲区写入数据
	if not isinstance(data, (bytes, bytearray, memoryview)):
		raise TypeError(f"data: expecting a bytes-like instance, "
						f"got {type(data).__name__}")
	if not data:
		return
	self._ssl_protocol._write_appdata(data)
```
### def can_write_eof
```python
def can_write_eof(self):
	return False
```
### def abort
```python
def abort(self):
	self._ssl_protocol._abort()
	self._closed = True
```
## class SSLProtocol
### 初始化
```python
class SSLProtocol(protocols.Protocol):
    # 使用SSL传入缓冲区和传出缓冲区在套接字上实现SSL。MemoryBIO对象

    def __init__(self, loop, app_protocol, sslcontext, waiter,
                 server_side=False, server_hostname=None,
                 call_connection_made=True,
                 ssl_handshake_timeout=None):
        if ssl is None:
            raise RuntimeError('stdlib ssl module not available')

        if ssl_handshake_timeout is None:
            ssl_handshake_timeout = constants.SSL_HANDSHAKE_TIMEOUT
        elif ssl_handshake_timeout <= 0:
            raise ValueError(
                f"ssl_handshake_timeout should be a positive number, "
                f"got {ssl_handshake_timeout}")

        # 创建一个ssl上下文只对客户端生效
        if not sslcontext:
            sslcontext = _create_transport_context(
                server_side, server_hostname)

        # 服务端标识
        self._server_side = server_side
        if server_hostname and not server_side:
            self._server_hostname = server_hostname
        else:
            self._server_hostname = None
        self._sslcontext = sslcontext
        # ssl额外消息，更多消息将在握手完成时设置
        self._extra = dict(sslcontext=sslcontext)

        # 应用数据写缓冲
        self._write_backlog = collections.deque()
        self._write_buffer_size = 0

        self._waiter = waiter
        self._loop = loop
        # 设置app侧的协议
        self._set_app_protocol(app_protocol)
        # app的传输
        self._app_transport = _SSLProtocolTransport(self._loop, self)
        # _SSLPipe 实例
        self._sslpipe = None
        self._session_established = False
        # 正在握手状态
        self._in_handshake = False
        # 正在关闭状态
        self._in_shutdown = False
        # 传输对象
        self._transport = None
        self._call_connection_made = call_connection_made
        self._ssl_handshake_timeout = ssl_handshake_timeout
```
### def _set_app_protocol
设置app侧的协议，并标记协议类型
```python
def _set_app_protocol(self, app_protocol):
	self._app_protocol = app_protocol
	self._app_protocol_is_buffer = \
		isinstance(app_protocol, protocols.BufferedProtocol)
```
### def _wakeup_waiter
唤醒被阻塞的协程
```python
def _wakeup_waiter(self, exc=None):
	if self._waiter is None:
		return
	if not self._waiter.cancelled():
		if exc is not None:
			self._waiter.set_exception(exc)
		else:
			self._waiter.set_result(None)
	self._waiter = None
```
### def connection_made
连接初始化时调用
```python
def connection_made(self, transport):
	# 设置传输
	self._transport = transport
	# 创建管道
	self._sslpipe = _SSLPipe(self._sslcontext,
							 self._server_side,
							 self._server_hostname)
	# 开始握手
	self._start_handshake()
```
### def connection_lost
传输连接丢失回调的方法
```python
def connection_lost(self, exc):
	# 调用app协议回调函数
	if self._session_established:
		self._session_established = False
		self._loop.call_soon(self._app_protocol.connection_lost, exc)
	else:
		# 很可能在SSL握手时发生了异常
		if self._app_transport is not None:
			self._app_transport._closed = True
	# 清理对象
	self._transport = None
	self._app_transport = None
	if getattr(self, '_handshake_timeout_handle', None):
		self._handshake_timeout_handle.cancel()
	# 向等待的协程发送异常
	self._wakeup_waiter(exc)
	self._app_protocol = None
	self._sslpipe = None
```
### def pause_writing
app协议暂停写入
```python
def pause_writing(self):
	self._app_protocol.pause_writing()
```
### def resume_writing
app协议恢复写入
```python
def resume_writing(self):
	self._app_protocol.resume_writing()
```
### def data_received
接收数据
```python
def data_received(self, data):
	# 根据协议的不同调用不同的方法接收数据
	if self._sslpipe is None:
		# transport closing, sslpipe is destroyed
		return

	try:
		ssldata, appdata = self._sslpipe.feed_ssldata(data)
	except Exception as e:
		self._fatal_error(e, 'SSL error in data received')
		return

	for chunk in ssldata:
		self._transport.write(chunk)

	for chunk in appdata:
		if chunk:
			try:
				if self._app_protocol_is_buffer:
					protocols._feed_data_to_buffered_proto(
						self._app_protocol, chunk)
				else:
					self._app_protocol.data_received(chunk)
			except Exception as ex:
				self._fatal_error(
					ex, 'application protocol failed to receive SSL data')
				return
		else:
			self._start_shutdown()
			break
```
### def eof_received
接收数据结束
```python
def eof_received(self):
	# 接收数据完成，判断是否需要关闭低级流
	try:
		if self._loop.get_debug():
			logger.debug("%r received EOF", self)

		self._wakeup_waiter(ConnectionResetError)

		if not self._in_handshake:
			keep_open = self._app_protocol.eof_received()
			if keep_open:
				logger.warning('returning true from eof_received() '
							   'has no effect when using ssl')
	finally:
		self._transport.close()
```
### def _get_extra_info
获取额外的信息
```python
def _get_extra_info(self, name, default=None):
	# 获取额外的数据
	if name in self._extra:
		return self._extra[name]
	elif self._transport is not None:
		return self._transport.get_extra_info(name, default)
	else:
		return default
```
### def _start_shutdown
开始关闭ssl
```python
def _start_shutdown(self):
	# 开始关闭ssl
	if self._in_shutdown:
		return
	if self._in_handshake:
		self._abort()
	else:
		self._in_shutdown = True
		# 空数据表示 结束握手
		self._write_appdata(b'')
```
### def _write_appdata
把app发送的数据保存到缓冲区
```python
def _write_appdata(self, data):
	# 写入app发送的数据
	self._write_backlog.append((data, 0))
	# 缓冲区大小增加
	self._write_buffer_size += len(data)
	# 进程写入可能
	self._process_write_backlog()
```
### def _start_handshake
ssl 开始握手
```python
def _start_handshake(self):
	# 开始 ssl 握手
	if self._loop.get_debug():
		logger.debug("%r starts SSL handshake", self)
		self._handshake_start_time = self._loop.time()
	else:
		self._handshake_start_time = None
	# 正在握手中
	self._in_handshake = True
	# 写入数据，1表示握手
	self._write_backlog.append((b'', 1))
	# 超时回调函数
	self._handshake_timeout_handle = \
		self._loop.call_later(self._ssl_handshake_timeout,
							  self._check_handshake_timeout)
	self._process_write_backlog()
```
### def _check_handshake_timeout
检查握手是否超时
```python
def _check_handshake_timeout(self):
	# 检查握手有没有完成
	if self._in_handshake is True:
		msg = (
			f"SSL handshake is taking longer than "
			f"{self._ssl_handshake_timeout} seconds: "
			f"aborting the connection"
		)
		self._fatal_error(ConnectionAbortedError(msg))
```
### def _on_handshake_complete
握手完成时调用的方法
```python
def _on_handshake_complete(self, handshake_exc):
	# 设置握手状态
	self._in_handshake = False
	# 取消握手的超时检测回调
	self._handshake_timeout_handle.cancel()

	sslobj = self._sslpipe.ssl_object
	try:
		if handshake_exc is not None:
			raise handshake_exc
		# 证书对
		peercert = sslobj.getpeercert()
	except Exception as exc:
		# 验证证书失败
		if isinstance(exc, ssl.CertificateError):
			msg = 'SSL handshake failed on verifying the certificate'
		else:
			msg = 'SSL handshake failed'
		self._fatal_error(exc, msg)
		return

	# 记录握手的时间
	if self._loop.get_debug():
		dt = self._loop.time() - self._handshake_start_time
		logger.debug("%r: SSL handshake took %.1f ms", self, dt * 1e3)

	# 添加握手后可用的额外信息，证书，加密方式
	self._extra.update(peercert=peercert,
					   cipher=sslobj.cipher(),
					   compression=sslobj.compression(),
					   ssl_object=sslobj,
					   )
	# 初始化协议，设置传输
	if self._call_connection_made:
		self._app_protocol.connection_made(self._app_transport)
	# 唤醒写
	self._wakeup_waiter()
	# 正在连接中的状态设置为真
	self._session_established = True
	# 不要直接调用
	self._loop.call_soon(self._process_write_backlog)
```
### def _process_write_backlog
把积压的数据发送出去
```python
def _process_write_backlog(self):
	# 把积压的数据发送
	if self._transport is None or self._sslpipe is None:
		return

	try:
		for i in range(len(self._write_backlog)):
			data, offset = self._write_backlog[0]
			# 如果有数据，通过管道发送
			if data:
				ssldata, offset = self._sslpipe.feed_appdata(data, offset)
			# 如果是建立握手，添加回调函数
			elif offset:
				ssldata = self._sslpipe.do_handshake(
					self._on_handshake_complete)
				offset = 1
			# 如果是断开握手
			else:
				ssldata = self._sslpipe.shutdown(self._finalize)
				offset = 1

			# 传输写入数据
			for chunk in ssldata:
				self._transport.write(chunk)

			if offset < len(data):
				self._write_backlog[0] = (data, offset)
				# 短写意味着写被阻塞在一个读上，如果被暂停，我们需要恢复他
				assert self._sslpipe.need_ssldata
				if self._transport._paused:
					self._transport.resume_reading()
				break

			# 待办事项的一块被完成了，我们需要删除它
			del self._write_backlog[0]
			self._write_buffer_size -= len(data)
	except Exception as exc:
		if self._in_handshake:
			# 异常将在握手完成时重新引发
			self._on_handshake_complete(exc)
		else:
			self._fatal_error(exc, 'Fatal error on SSL transport')
```
### def _fatal_error
处理异常
```python
def _fatal_error(self, exc, message='Fatal error on transport'):
	# 处理异常
	if isinstance(exc, OSError):
		if self._loop.get_debug():
			logger.debug("%r: %s", self, message, exc_info=True)
	else:
		self._loop.call_exception_handler({
			'message': message,
			'exception': exc,
			'transport': self._transport,
			'protocol': self,
		})
	if self._transport:
		self._transport._force_close(exc)
```
### def _finalize
```python
def _finalize(self):
	# 关闭传输
	self._sslpipe = None

	if self._transport is not None:
		self._transport.close()
```
### def _abort
```python
def _abort(self):
	# 强制断开传输
	try:
		if self._transport is not None:
			self._transport.abort()
	finally:
		self._finalize()
```
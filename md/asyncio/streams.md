[TOC]
# asyncio 之 streams.py
## 摘要
## class IncompleteReadError
```python
class IncompleteReadError(EOFError):
    """
    Incomplete read error. Attributes:

    - partial: read bytes string before the end of stream was reached
    - expected: total number of expected bytes (or None if unknown)
    """
    def __init__(self, partial, expected):
        super().__init__(f'{len(partial)} bytes read on a total of '
                         f'{expected!r} expected bytes')
        self.partial = partial
        self.expected = expected

    def __reduce__(self):
        return type(self), (self.partial, self.expected)
```
## class LimitOverrunError
```python
class LimitOverrunError(Exception):
    """Reached the buffer limit while looking for a separator.

    Attributes:
    - consumed: total number of to be consumed bytes.
    """
    def __init__(self, message, consumed):
        super().__init__(message)
        self.consumed = consumed

    def __reduce__(self):
        return type(self), (self.args[0], self.consumed)
```
## async def open_connection
一个用于创建连接的包装器
```python
async def open_connection(host=None, port=None, *,
                          loop=None, limit=_DEFAULT_LIMIT, **kwds):
	# 事件循环
    if loop is None:
        loop = events.get_event_loop()
	# 读取流
    reader = StreamReader(limit=limit, loop=loop)
	# 读取协议
    protocol = StreamReaderProtocol(reader, loop=loop)
	# 根据协议和地址端口创建传输
    transport, _ = await loop.create_connection(
        lambda: protocol, host, port, **kwds)
	# 根据传输和协议创建的写入流
    writer = StreamWriter(transport, protocol, reader, loop)
	# 返回 读取，写入
    return reader, writer
```
## async def start_server
开启一个套接字服务器，为每个连接的客户端创建回调函数，返回一个服务器对象。
```python
async def start_server(client_connected_cb, host=None, port=None, *,
                       loop=None, limit=_DEFAULT_LIMIT, **kwds):
    """
	`client_connected_cb` 接受两个参数，读取流，写入流，可以是一个回调函数，也可以是一个协程。
    """
	# 事件循环
    if loop is None:
        loop = events.get_event_loop()

    def factory():
		# 创建读取流
        reader = StreamReader(limit=limit, loop=loop)
		# 读取协议
        protocol = StreamReaderProtocol(reader, client_connected_cb,
                                        loop=loop)
        return protocol
	# 创建一个套接字服务器
    return await loop.create_server(factory, host, port, **kwds)
```
## UNIX 套接字专用
```python
if hasattr(socket, 'AF_UNIX'):
    # UNIX Domain Sockets are supported on this platform
	...
```
### async def open_unix_connection
```python
async def open_unix_connection(path=None, *,
							   loop=None, limit=_DEFAULT_LIMIT, **kwds):
	"""Similar to `open_connection` but works with UNIX Domain Sockets."""
	if loop is None:
		loop = events.get_event_loop()
	reader = StreamReader(limit=limit, loop=loop)
	protocol = StreamReaderProtocol(reader, loop=loop)
	transport, _ = await loop.create_unix_connection(
		lambda: protocol, path, **kwds)
	writer = StreamWriter(transport, protocol, reader, loop)
	return reader, writer
```
### async def start_unix_server
```python
async def start_unix_server(client_connected_cb, path=None, *,
							loop=None, limit=_DEFAULT_LIMIT, **kwds):
	"""Similar to `start_server` but works with UNIX Domain Sockets."""
	if loop is None:
		loop = events.get_event_loop()

	def factory():
		reader = StreamReader(limit=limit, loop=loop)
		protocol = StreamReaderProtocol(reader, client_connected_cb,
										loop=loop)
		return protocol

	return await loop.create_unix_server(factory, path, **kwds)
```
## class FlowControlMixin
可重用流控制逻辑
### 初始化
```python
class FlowControlMixin(protocols.Protocol):
    def __init__(self, loop=None):
        if loop is None:
            self._loop = events.get_event_loop()
        else:
            self._loop = loop
		# 暂停状态
        self._paused = False
		# 消耗程序
        self._drain_waiter = None
        self._connection_lost = False
```
### def pause_writing
```python
def pause_writing(self):
	assert not self._paused
	self._paused = True
	if self._loop.get_debug():
		logger.debug("%r pauses writing", self)
```
### def resume_writing
```python
def resume_writing(self):
	assert self._paused
	self._paused = False
	if self._loop.get_debug():
		logger.debug("%r resumes writing", self)

	waiter = self._drain_waiter
	if waiter is not None:
		self._drain_waiter = None
		if not waiter.done():
			waiter.set_result(None)
```
### def connection_lost
```python
def connection_lost(self, exc):
	self._connection_lost = True
	# Wake up the writer if currently paused.
	if not self._paused:
		return
	waiter = self._drain_waiter
	if waiter is None:
		return
	self._drain_waiter = None
	if waiter.done():
		return
	if exc is None:
		waiter.set_result(None)
	else:
		waiter.set_exception(exc)
```
### async def _drain_helper
```python
async def _drain_helper(self):
	if self._connection_lost:
		raise ConnectionResetError('Connection lost')
	if not self._paused:
		return
	waiter = self._drain_waiter
	assert waiter is None or waiter.cancelled()
	waiter = self._loop.create_future()
	self._drain_waiter = waiter
	await waiter
```
## class StreamReaderProtocol
协议和流读取的适配器类
### 初始化
```python
class StreamReaderProtocol(FlowControlMixin, protocols.Protocol):
    def __init__(self, stream_reader, client_connected_cb=None, loop=None):
        super().__init__(loop=loop)
		# 流读取对象
        self._stream_reader = stream_reader
		# 流写入对象
        self._stream_writer = None
		# 连接回调函数
        self._client_connected_cb = client_connected_cb
		# 有没有ssl
        self._over_ssl = False
		# 关闭对象
        self._closed = self._loop.create_future()
```
### def connection_made
```python
def connection_made(self, transport):
	# 给流读取对象设置传输
	self._stream_reader.set_transport(transport)
	# 是否使用了 ssl 协议
	self._over_ssl = transport.get_extra_info('sslcontext') is not None
	if self._client_connected_cb is not None:
		# 创建一个流写入对象
		self._stream_writer = StreamWriter(transport, self,
										   self._stream_reader,
										   self._loop)
		# 调用连接回调
		res = self._client_connected_cb(self._stream_reader,
										self._stream_writer)
		# 如果连接回调是一个协程，包装成一个任务
		if coroutines.iscoroutine(res):
			self._loop.create_task(res)
```
### def connection_lost
```python
def connection_lost(self, exc):
	if self._stream_reader is not None:
		# 断开流读取连接或者发送异常
		if exc is None:
			self._stream_reader.feed_eof()
		else:
			self._stream_reader.set_exception(exc)
	# 给关闭对象设置结果或者异常
	if not self._closed.done():
		if exc is None:
			self._closed.set_result(None)
		else:
			self._closed.set_exception(exc)
	# 调用父类的连接丢失
	super().connection_lost(exc)
	# 重置流读取/流写入对象
	self._stream_reader = None
	self._stream_writer = None
```
### def data_received
接收数据
```python
def data_received(self, data):
	self._stream_reader.feed_data(data)
```
### def eof_received
接收数据结束
```python
def eof_received(self):
	self._stream_reader.feed_eof()
	if self._over_ssl:
		# Prevent a warning in SSLProtocol.eof_received:
		# "returning true from eof_received()
		# has no effect when using ssl"
		return False
	return True
```
### def __del__
删除
```python
def __del__(self):
	# Prevent reports about unhandled exceptions.
	# Better than self._closed._log_traceback = False hack
	closed = self._closed
	if closed.done() and not closed.cancelled():
		closed.exception()
```
## class StreamWriter
写入流，包装传输
### 初始化
```python
class StreamWriter:
    def __init__(self, transport, protocol, reader, loop):
		# 初始化传输和协议
        self._transport = transport
        self._protocol = protocol
		# 读取必须为空或者是读取流
        assert reader is None or isinstance(reader, StreamReader)
        self._reader = reader
        self._loop = loop
```
### def transport
```python
@property
def transport(self):
	return self._transport
```
### 调用传输的方法
```python
def write(self, data):
	self._transport.write(data)

def writelines(self, data):
	self._transport.writelines(data)

def write_eof(self):
	return self._transport.write_eof()

def can_write_eof(self):
	return self._transport.can_write_eof()

def close(self):
	return self._transport.close()

def is_closing(self):
	return self._transport.is_closing()

def get_extra_info(self, name, default=None):
	return self._transport.get_extra_info(name, default)
```
### async def wait_closed
等待协议关闭
```python
async def wait_closed(self):
    await self._protocol._closed
```
### async def drain
刷新写缓冲区
```python
async def drain(self):
	"""
	  w.write(data)
	  await w.drain()
	"""
	# 查看读取流是否出现了异常
	if self._reader is not None:
		exc = self._reader.exception()
		if exc is not None:
			raise exc
	# 如果传输已经关闭
	if self._transport.is_closing():
		# Yield to the event loop so connection_lost() may be called.
		# Without this, _drain_helper() would return
		# immediately, and code that calls
		#     write(...); await drain()
		# in a loop would never call connection_lost(), so it
		# would not see an error when the socket is closed.
		await sleep(0, loop=self._loop)
	await self._protocol._drain_helper()
```
## class 
### 初始化
```python
class StreamReader:
    def __init__(self, limit=_DEFAULT_LIMIT, loop=None):
        # The line length limit is  a security feature;
        # it also doubles as half the buffer limit.

        if limit <= 0:
            raise ValueError('Limit cannot be <= 0')

        self._limit = limit
        if loop is None:
            self._loop = events.get_event_loop()
        else:
            self._loop = loop
		# 缓冲区
        self._buffer = bytearray()
		# 完成标识
        self._eof = False
		# A future used by _wait_for_data()
        self._waiter = None
        self._exception = None
		# 传输
        self._transport = None
		# 暂停状态
        self._paused = False
```
### def
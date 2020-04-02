[TOC]
## 摘要
文件实现了创建连接、创建server的方法。和流控制逻辑(用于协议)类、流读取协议类、流读取类和流写入类。流读取和流写入类的实例化对象是返回给用户使用的一对对象，他们使用相同的协议和传输。
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
一个用于创建连接的包装器，返回读取流和写入流
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
流量控制逻辑
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
		# 等待写入数据的协程
        self._drain_waiter = None
        self._connection_lost = False
```
### def pause_writing
暂停协议，控制写入流停止向传输写入数据。
```python
def pause_writing(self):
	assert not self._paused
	self._paused = True
	if self._loop.get_debug():
		logger.debug("%r pauses writing", self)
```
### def resume_writing
恢复协议，通知写入流可以向传输写入数据
```python
def resume_writing(self):
	assert self._paused
	self._paused = False
	if self._loop.get_debug():
		logger.debug("%r resumes writing", self)
	# 等待写入数据的协程
	waiter = self._drain_waiter
	if waiter is not None:
		self._drain_waiter = None
		if not waiter.done():
			waiter.set_result(None)
```
### def connection_lost
连接丢失回调函数
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
写入流向传输写入数据的辅助方法，控制写入流的数据传输。
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
流读取协议类，用来控制写入流、读取流和传输之见的交互。
### 初始化
```python
class StreamReaderProtocol(FlowControlMixin, protocols.Protocol):
    def __init__(self, stream_reader, client_connected_cb=None, loop=None):
        super().__init__(loop=loop)
		# 流读取对象
        self._stream_reader = stream_reader
		# 流写入对象
        self._stream_writer = None
		# 连接回调函数，创建服务器协议对象需要设置的参数。
        self._client_connected_cb = client_connected_cb
		# 有没有ssl
        self._over_ssl = False
		# 关闭对象
        self._closed = self._loop.create_future()
```
### def connection_made
协议初始化，初始化传输时被调用。
```python
def connection_made(self, transport):
	# 给流读取对象设置传输
	self._stream_reader.set_transport(transport)
	# 是否使用了 ssl 协议
	self._over_ssl = transport.get_extra_info('sslcontext') is not None
	# 如果是服务器协议对象
	if self._client_connected_cb is not None:
		# 创建一个流写入对象
		self._stream_writer = StreamWriter(transport, self,
										   self._stream_reader,
										   self._loop)
		# 服务器连接的回调函数，用来接收连接的数据，并返回结果。
		res = self._client_connected_cb(self._stream_reader,
										self._stream_writer)
		# 如果连接回调是一个协程，包装成一个任务
		if coroutines.iscoroutine(res):
			self._loop.create_task(res)
```
### def connection_lost
连接丢失时调用的方法
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
被传输调用，发送数据到读取流缓冲中。
```python
def data_received(self, data):
	self._stream_reader.feed_data(data)
```
### def eof_received
被传输调用，套接字数据接收完毕。
```python
def eof_received(self):
	# 通知读取流数据接收结束。
	self._stream_reader.feed_eof()
	if self._over_ssl:
		return False
	return True
```
### def __del__
删除时清理方法，消费掉异常。
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
写入流通过协议调用传输的方法发送数据。
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
刷新传输的缓冲区，防止背压。
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
		# 放到下一次事件循环中执行，否则可能错过连接丢失的调用，不会看到关闭异常。
		await sleep(0, loop=self._loop)
	await self._protocol._drain_helper()
```
## class StreamReader
### 初始化
```python
class StreamReader:
    def __init__(self, limit=_DEFAULT_LIMIT, loop=None):
        if limit <= 0:
            raise ValueError('Limit cannot be <= 0')
		# 读取流缓冲区大小
        self._limit = limit
        if loop is None:
            self._loop = events.get_event_loop()
        else:
            self._loop = loop
		# 缓冲区方法
        self._buffer = bytearray()
		# 接受数据完成标识
        self._eof = False
		# 等待从读取流接收数据的用户协程
        self._waiter = None
        self._exception = None
		# 传输
        self._transport = None
		# 缓冲区控制属性
        self._paused = False
```
### def exception
返回异常
```python
def exception(self):
	return self._exception
```
### def set_exception
设置异常，并唤醒等待读取数据的用户协程
```python
def set_exception(self, exc):
	self._exception = exc

	waiter = self._waiter
	if waiter is not None:
		self._waiter = None
		if not waiter.cancelled():
			waiter.set_exception(exc)
```
### def _wakeup_waiter
唤醒等待读取数据的用户协程
```python
def _wakeup_waiter(self):
	"""Wakeup read*() functions waiting for data or EOF."""
	waiter = self._waiter
	if waiter is not None:
		self._waiter = None
		if not waiter.cancelled():
			waiter.set_result(None)
```
### def set_transport
被协议调用，设置传输。
```python
def set_transport(self, transport):
	assert self._transport is None, 'Transport already set'
	self._transport = transport
```
### def _maybe_resume_transport
通知传输可以读取套接字数据到当前缓冲区。
```python
def _maybe_resume_transport(self):
	if self._paused and len(self._buffer) <= self._limit:
		self._paused = False
		self._transport.resume_reading()
```
### def feed_eof
传输数据读取完毕，设置结束标识，唤醒用户协程。
```python
def feed_eof(self):
	self._eof = True
	self._wakeup_waiter()
```
### def at_eof
传输读取数据完毕并且用户已经把数据接受完毕返回真
```python
def at_eof(self):
	"""Return True if the buffer is empty and 'feed_eof' was called."""
	return self._eof and not self._buffer
```
### def feed_data
把传输读取到的数据放到缓冲区中，并唤醒读取数据的用户协程
```python
def feed_data(self, data):
	assert not self._eof, 'feed_data after feed_eof'

	if not data:
		return

	self._buffer.extend(data)
	self._wakeup_waiter()

	# 缓冲区大小超出限制，暂停传输读取数据，设置暂停状态为真
	if (self._transport is not None and
			not self._paused and
			len(self._buffer) > 2 * self._limit):
		try:
			self._transport.pause_reading()
		except NotImplementedError:
			# 传输不能暂停，设置传输为None，这样就不会一直尝试
			self._transport = None
		else:
			self._paused = True
```
### def _wait_for_data
等待从流中读取数据
```python
async def _wait_for_data(self, func_name):
	# 如果等待读取的协程已经存在抛出异常
	if self._waiter is not None:
		raise RuntimeError(
			f'{func_name}() called while another coroutine is '
			f'already waiting for incoming data')

	assert not self._eof, '_wait_for_data after EOF'

	# 暂停时等待数据会导致死锁
	# This is essential for readexactly(n) for case when n > self._limit.
	if self._paused:
		self._paused = False
		self._transport.resume_reading()

	self._waiter = self._loop.create_future()
	try:
		await self._waiter
	finally:
		self._waiter = None
```
### async def readline
读取一行数据
```python
async def readline(self):
	sep = b'\n'
	seplen = len(sep)
	# 读取数据直到分隔符出现
	try:
		line = await self.readuntil(sep)
	except IncompleteReadError as e:
		return e.partial
	except LimitOverrunError as e:
		if self._buffer.startswith(sep, e.consumed):
			del self._buffer[:e.consumed + seplen]
		else:
			self._buffer.clear()
		self._maybe_resume_transport()
		raise ValueError(e.args[0])
	return line
```
### async def readuntil
读取数据直到遇到分隔符或这抛出异常。
```python
async def readuntil(self, separator=b'\n'):
	# 分隔符的长度
	seplen = len(separator)
	if seplen == 0:
		raise ValueError('Separator should be at least one-byte string')

	if self._exception is not None:
		raise self._exception

	# 偏移量
	offset = 0

	# 循环直到我们在缓冲区找到分隔符或者出现EOF或者超过limit限制。
	while True:
		buflen = len(self._buffer)

		# 检查缓冲区是否有足够的数据去寻找分隔符
		if buflen - offset >= seplen:
			isep = self._buffer.find(separator, offset)
			# 找到了分隔符，跳出循环。
			if isep != -1:
				break

			# 设置新的偏移量
			offset = buflen + 1 - seplen
			# 偏移量大于限制抛出异常，数据还在缓冲区可以再次读取。
			if offset > self._limit:
				raise LimitOverrunError(
					'Separator is not found, and chunk exceed the limit',
					offset)

		# 如果传输数据已经结束仍没有找到标识符，把缓冲区的数据随异常一起抛出，清空缓冲区。
		if self._eof:
			chunk = bytes(self._buffer)
			self._buffer.clear()
			raise IncompleteReadError(chunk, None)

		# 等待缓冲区数据更新
		await self._wait_for_data('readuntil')

	if isep > self._limit:
		raise LimitOverrunError(
			'Separator is found, but chunk is longer than limit', isep)
	# 找到标识符，清除缓冲区部分数据，返回结果。
	chunk = self._buffer[:isep + seplen]
	del self._buffer[:isep + seplen]
	self._maybe_resume_transport()
	return bytes(chunk)
```
### async def read
读取指定字节的长度的数据。默认读取所有，不受limit限制。
```python
async def read(self, n=-1):
	if self._exception is not None:
		raise self._exception

	if n == 0:
		return b''

	if n < 0:
		blocks = []
		# 循环读取数据直到结束。
		while True:
			block = await self.read(self._limit)
			if not block:
				break
			blocks.append(block)
		return b''.join(blocks)

	# 等待缓冲区数据更新
	if not self._buffer and not self._eof:
		await self._wait_for_data('read')

	data = bytes(self._buffer[:n])
	del self._buffer[:n]

	self._maybe_resume_transport()
	return data
```
### async def readexactly
准确的读取指定字节长度的数据。如果长度不够则缓冲区数据会随异常一起被抛出。
```python
async def readexactly(self, n):
	if n < 0:
		raise ValueError('readexactly size can not be less than zero')

	if self._exception is not None:
		raise self._exception

	if n == 0:
		return b''

	while len(self._buffer) < n:
		if self._eof:
			incomplete = bytes(self._buffer)
			self._buffer.clear()
			raise IncompleteReadError(incomplete, n)

		await self._wait_for_data('readexactly')

	if len(self._buffer) == n:
		data = bytes(self._buffer)
		self._buffer.clear()
	else:
		data = bytes(self._buffer[:n])
		del self._buffer[:n]
	self._maybe_resume_transport()
	return data
```
### def __aiter__
```python
def __aiter__(self):
	return self
```
### def __anext__
```python
async def __anext__(self):
	val = await self.readline()
	if val == b'':
		raise StopAsyncIteration
	return val
```
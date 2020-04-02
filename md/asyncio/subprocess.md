[TOC]
## 摘要
文件中实现了两个类，SubprocessStreamProtocol: 子进程协议，控制子进程传输的数据读写。Process: 进程包装类，实例化需要传递两个参数分别是: tansport(子进程传输对象)、 protocol(子进程协议对象)。实例化之后抛出给用户使用，有读取和写入等接口。
## class SubprocessStreamProtocol
### 初始化
```python
class SubprocessStreamProtocol(streams.FlowControlMixin,
                               protocols.SubprocessProtocol):
    """Like StreamReaderProtocol, but for a subprocess."""
    def __init__(self, limit, loop):
        super().__init__(loop=loop)
        # 缓冲区限制大小
        self._limit = limit
        # 进程管道被包装之后的三个流对象，用于读取和写入数据
        self.stdin = self.stdout = self.stderr = None
        # 子进程传输
        self._transport = None
        # 进程退出状态
        self._process_exited = False
		# 管道的文件描述符列表
        self._pipe_fds = []
```
### def connection_made
子进程传输实例化时被调用，协议初始化方法
```python
def connection_made(self, transport):
	# 设置子进程协议的子进程传输
	self._transport = transport
	# 输出管道的传输
	stdout_transport = transport.get_pipe_transport(1)
	if stdout_transport is not None:
		# 输出流
		self.stdout = streams.StreamReader(limit=self._limit,
										   loop=self._loop)
		self.stdout.set_transport(stdout_transport)
		self._pipe_fds.append(1)
	# 错误管道的传输
	stderr_transport = transport.get_pipe_transport(2)
	if stderr_transport is not None:
		# 错误流
		self.stderr = streams.StreamReader(limit=self._limit,
										   loop=self._loop)
		self.stderr.set_transport(stderr_transport)
		self._pipe_fds.append(2)
	# 输入管道的传输
	stdin_transport = transport.get_pipe_transport(0)
	if stdin_transport is not None:
		# 输入流
		self.stdin = streams.StreamWriter(stdin_transport,
										  protocol=self,
										  reader=None,
										  loop=self._loop)
```
### def pipe_data_received
从管道中接收数据(out or err)
```python
def pipe_data_received(self, fd, data):
	# 选择管道
	if fd == 1:
		reader = self.stdout
	elif fd == 2:
		reader = self.stderr
	else:
		reader = None
	# 接收数据
	if reader is not None:
		reader.feed_data(data)
```
### def pipe_connection_lost
管道连接丢失的回调方法
```python
def pipe_connection_lost(self, fd, exc):
	# 根据管道类型选择
	if fd == 0:
		pipe = self.stdin
		if pipe is not None:
			pipe.close()
		self.connection_lost(exc)
		return
	if fd == 1:
		reader = self.stdout
	elif fd == 2:
		reader = self.stderr
	else:
		reader = None
	# 如果是输出管道，设置结束标识符
	if reader is not None:
		if exc is None:
			reader.feed_eof()
		else:
			reader.set_exception(exc)

	if fd in self._pipe_fds:
		self._pipe_fds.remove(fd)
	self._maybe_close_transport()
```
### def process_exited
进程退出
```python
def process_exited(self):
	# 设置协议的状态
	self._process_exited = True
	# 尝试关闭传输
	self._maybe_close_transport()
```
### def _maybe_close_transport
尝试关闭传输
```python
def _maybe_close_transport(self):
	# 关闭传输
	if len(self._pipe_fds) == 0 and self._process_exited:
		self._transport.close()
		self._transport = None
```
## class Process
### 初始化
```python
class Process:
    def __init__(self, transport, protocol, loop):
		# 子进程传输
        self._transport = transport
		# 子进程协议
        self._protocol = protocol
        self._loop = loop
		# 子进程输入流
        self.stdin = protocol.stdin
		# 子进程输出流
        self.stdout = protocol.stdout
		# 子进程错误流
        self.stderr = protocol.stderr
        # 传输的进程id
        self.pid = transport.get_pid()
```
### def returncode
返回进程退出码
```python
@property
def returncode(self):
	return self._transport.get_returncode()
```
### async def wait
等待子进程传输退出并返回进程退出码
```python
async def wait(self):
	return await self._transport._wait()
```
### def send_signal
通过子进程传输发送一个信号
```python
def send_signal(self, signal):
	self._transport.send_signal(signal)
```
### def terminate
终止传输
```python
def terminate(self):
	self._transport.terminate()
```
### def kill
kill 传输
```python
def kill(self):
	self._transport.kill()
```
### async def _feed_stdin
通过进程输入流向进程发送数据
```python
async def _feed_stdin(self, input):
	debug = self._loop.get_debug()
	self.stdin.write(input)
	if debug:
		logger.debug(
			'%r communicate: feed stdin (%s bytes)', self, len(input))
	try:
		await self.stdin.drain()
	except (BrokenPipeError, ConnectionResetError) as exc:
		# communicate() ignores BrokenPipeError and ConnectionResetError
		if debug:
			logger.debug('%r communicate: stdin got %r', self, exc)

	if debug:
		logger.debug('%r communicate: close stdin', self)
	self.stdin.close()
```
### async def _noop
```python
async def _noop(self):
	return None
```
### async def _read_stream
从管道中读取数据
```python
async def _read_stream(self, fd):
	transport = self._transport.get_pipe_transport(fd)
	if fd == 2:
		stream = self.stderr
	else:
		assert fd == 1
		stream = self.stdout
	if self._loop.get_debug():
		name = 'stdout' if fd == 1 else 'stderr'
		logger.debug('%r communicate: read %s', self, name)
	output = await stream.read()
	if self._loop.get_debug():
		name = 'stdout' if fd == 1 else 'stderr'
		logger.debug('%r communicate: close %s', self, name)
	transport.close()
	return output
```
### async def communicate
通过输入管道发送数据然后等待从输出管道接收数据
```python
async def communicate(self, input=None):
	# 发送数据，等待结果
	if input is not None:
		stdin = self._feed_stdin(input)
	else:
		stdin = self._noop()
	if self.stdout is not None:
		stdout = self._read_stream(1)
	else:
		stdout = self._noop()
	if self.stderr is not None:
		stderr = self._read_stream(2)
	else:
		stderr = self._noop()
	# 等待三个协成完成并返回结果
	stdin, stdout, stderr = await tasks.gather(stdin, stdout, stderr, loop=self._loop)
	await self.wait()
	return (stdout, stderr)
```
## async def create_subprocess_shell
创建一个子进程执行命令
```python
async def create_subprocess_shell(cmd, stdin=None, stdout=None, stderr=None,
                                  loop=None, limit=streams._DEFAULT_LIMIT,
                                  **kwds):
    if loop is None:
        loop = events.get_event_loop()
    protocol_factory = lambda: SubprocessStreamProtocol(limit=limit,
                                                        loop=loop)
    transport, protocol = await loop.subprocess_shell(
        protocol_factory,
        cmd, stdin=stdin, stdout=stdout,
        stderr=stderr, **kwds)
    return Process(transport, protocol, loop)
```
## async def create_subprocess_exec
创建一个子进程执行程序
```python
async def create_subprocess_exec(program, *args, stdin=None, stdout=None,
                                 stderr=None, loop=None,
                                 limit=streams._DEFAULT_LIMIT, **kwds):
    if loop is None:
        loop = events.get_event_loop()
    protocol_factory = lambda: SubprocessStreamProtocol(limit=limit,
                                                        loop=loop)
    transport, protocol = await loop.subprocess_exec(
        protocol_factory,
        program, *args,
        stdin=stdin, stdout=stdout,
        stderr=stderr, **kwds)
    return Process(transport, protocol, loop)
```
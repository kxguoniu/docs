[TOC]
# asyncio 之 base_subprocess.py
## 摘要

## class BaseSubprocessTransport
### 初始化
```python
class BaseSubprocessTransport(transports.SubprocessTransport):

    def __init__(self, loop, protocol, args, shell,
                 stdin, stdout, stderr, bufsize,
                 waiter=None, extra=None, **kwargs):
        super().__init__(extra)
        self._closed = False
        self._protocol = protocol
        self._loop = loop
        self._proc = None
        self._pid = None
        self._returncode = None
        self._exit_waiters = []
        self._pending_calls = collections.deque()
        self._pipes = {}
        self._finished = False

        if stdin == subprocess.PIPE:
            self._pipes[0] = None
        if stdout == subprocess.PIPE:
            self._pipes[1] = None
        if stderr == subprocess.PIPE:
            self._pipes[2] = None

        # 创建子进程，设置属性
        try:
            self._start(args=args, shell=shell, stdin=stdin, stdout=stdout,
                        stderr=stderr, bufsize=bufsize, **kwargs)
        except:
            self.close()
            raise
        # 进程id
        self._pid = self._proc.pid
        self._extra['subprocess'] = self._proc

        if self._loop.get_debug():
            if isinstance(args, (bytes, str)):
                program = args
            else:
                program = args[0]
            logger.debug('process %r created: pid %s',
                         program, self._pid)

        self._loop.create_task(self._connect_pipes(waiter))
```
### def _start
子类重写的方法
```python
def _start(self, args, shell, stdin, stdout, stderr, bufsize, **kwargs):
	raise NotImplementedError
```
### def set_protocol
设置协议
```python
def set_protocol(self, protocol):
	self._protocol = protocol
```
### def get_protocol
获取协议
```python
def get_protocol(self):
	return self._protocol
```
### def is_closing
获取进程传输的状态
```python
def is_closing(self):
	return self._closed
```
### def close
```python
def close(self):
	if self._closed:
		return
	self._closed = True

	for proto in self._pipes.values():
		if proto is None:
			continue
		proto.pipe.close()

	if (self._proc is not None and
			# has the child process finished?
			self._returncode is None and
			# the child process has finished, but the
			# transport hasn't been notified yet?
			self._proc.poll() is None):

		if self._loop.get_debug():
			logger.warning('Close running child process: kill %r', self)

		try:
			self._proc.kill()
		except ProcessLookupError:
			pass
```
 ### def get_pid
 获取传输的进程id
 ```python
def get_pid(self):
	return self._pid
```
### def get_returncode
获取进程的退出码
```python
def get_returncode(self):
	return self._returncode
```
### def get_pipe_transport
获取进程指定的管道(in,out,err)
```python
def get_pipe_transport(self, fd):
	if fd in self._pipes:
		return self._pipes[fd].pipe
	else:
		return None
```
### def _check_proc
```python
def _check_proc(self):
	if self._proc is None:
		raise ProcessLookupError()
```
### def send_signal
向进程发送信号
```python
def send_signal(self, signal):
	self._check_proc()
	self._proc.send_signal(signal)
```
### def terminate
终止进程
```python
def terminate(self):
	self._check_proc()
	self._proc.terminate()
```
### def kill
杀掉进程
```python
def kill(self):
	self._check_proc()
	self._proc.kill()
```
### async def _connect_pipes
使用进程管道协议包装进程的三个管道
```python
async def _connect_pipes(self, waiter):
	try:
		proc = self._proc
		loop = self._loop

		if proc.stdin is not None:
			_, pipe = await loop.connect_write_pipe(
				lambda: WriteSubprocessPipeProto(self, 0),
				proc.stdin)
			self._pipes[0] = pipe

		if proc.stdout is not None:
			_, pipe = await loop.connect_read_pipe(
				lambda: ReadSubprocessPipeProto(self, 1),
				proc.stdout)
			self._pipes[1] = pipe

		if proc.stderr is not None:
			_, pipe = await loop.connect_read_pipe(
				lambda: ReadSubprocessPipeProto(self, 2),
				proc.stderr)
			self._pipes[2] = pipe

		assert self._pending_calls is not None

		loop.call_soon(self._protocol.connection_made, self)
		for callback, data in self._pending_calls:
			loop.call_soon(callback, *data)
		self._pending_calls = None
	except Exception as exc:
		if waiter is not None and not waiter.cancelled():
			waiter.set_exception(exc)
	else:
		if waiter is not None and not waiter.cancelled():
			waiter.set_result(None)
```
### def _call
添加进程管道包装后的回调函数或者立即调用
```python
def _call(self, cb, *data):
	# 进程管道还没有被包装，添加进列表等待调用
	if self._pending_calls is not None:
		self._pending_calls.append((cb, data))
	# 管道已经包装，立即调用
	else:
		self._loop.call_soon(cb, *data)
```
### def _pipe_connection_lost
添加管道连接丢失回调函数
```python
def _pipe_connection_lost(self, fd, exc):
	self._call(self._protocol.pipe_connection_lost, fd, exc)
	self._try_finish()
```
### def _pipe_data_received
在管道包装后调用管道接收数据
```python
def _pipe_data_received(self, fd, data):
	self._call(self._protocol.pipe_data_received, fd, data)
```
### def _process_exited
进程退出时被调用的方法，设置进程退出码
```python
def _process_exited(self, returncode):
	assert returncode is not None, returncode
	assert self._returncode is None, self._returncode
	if self._loop.get_debug():
		logger.info('%r exited with return code %r', self, returncode)
	self._returncode = returncode
	if self._proc.returncode is None:
		# asyncio uses a child watcher: copy the status into the Popen
		# object. On Python 3.6, it is required to avoid a ResourceWarning.
		self._proc.returncode = returncode
	self._call(self._protocol.process_exited)
	self._try_finish()

	# wake up futures waiting for wait()
	for waiter in self._exit_waiters:
		if not waiter.cancelled():
			waiter.set_result(returncode)
	self._exit_waiters = None
```
### async def _wait
阻塞直到进程退出并返回退出码
```python
async def _wait(self):
	if self._returncode is not None:
		return self._returncode

	waiter = self._loop.create_future()
	self._exit_waiters.append(waiter)
	return await waiter
```
### def _try_finish
在进程退出后尝试调用连接丢失方法
```python
def _try_finish(self):
	assert not self._finished
	if self._returncode is None:
		return
	# 进程结束后等待所有的管道关闭，设置属性，调用连接丢失
	if all(p is not None and p.disconnected
		   for p in self._pipes.values()):
		self._finished = True
		self._call(self._call_connection_lost, None)
```
### def _call_connection_lost
进程关闭后被调用的方法
```python
def _call_connection_lost(self, exc):
	try:
		self._protocol.connection_lost(exc)
	finally:
		self._loop = None
		self._proc = None
		self._protocol = None
```
## class WriteSubprocessPipeProto
### 初始化
```python
class WriteSubprocessPipeProto(protocols.BaseProtocol):

    def __init__(self, proc, fd):
        self.proc = proc
        self.fd = fd
        self.pipe = None
        self.disconnected = False
```
### 协议方法
```python
def connection_made(self, transport):
	self.pipe = transport

def connection_lost(self, exc):
	self.disconnected = True
	self.proc._pipe_connection_lost(self.fd, exc)
	self.proc = None

def pause_writing(self):
	self.proc._protocol.pause_writing()

def resume_writing(self):
	self.proc._protocol.resume_writing()
```
## class ReadSubprocessPipeProto
```python
class ReadSubprocessPipeProto(WriteSubprocessPipeProto,
                              protocols.Protocol):

    def data_received(self, data):
        self.proc._pipe_data_received(self.fd, data)
```
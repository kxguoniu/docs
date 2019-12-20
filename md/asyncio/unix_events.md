[TOC]
# asyncio 之 unix_events.py
## 摘要
## class
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
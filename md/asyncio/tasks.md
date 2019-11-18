# asyncio 之 tasks.py
## Task
### 类变量
示例对象的默认属性
```python
# 包含所有 task 的弱引用集合
_all_tasks = weakref.WeakSet()

# 记录所有 event loops 正在执行的 task
_current_tasks = {}

# task 在挂起状态时被销毁记录日志
_log_destroy_pending = True
```
### 获取 loop 当前正在执行的 task
```python
@classmethod
def current_task(cls, loop=None):
    # 如果没有 loop 获取默认的 loop
    if loop is None:
        loop = events.get_event_loop()
    # 返回 loop 正在运行的 task
    return cls._current_tasks.get(loop)
```
### 获取 loop 的所有 task
```python
@classmethod
def all_tasks(cls, loop=None):
    # 赋予默认的 loop
    if loop is None:
        loop = events.get_event_loop()
    return {t for t in cls._all_tasks if t._loop is loop}
```
### 初始化 task 对象
```python
def __init__(self, coro, *, loop=None):
	# 初始化参数必须是一个协程对象
    assert coroutines.iscoroutine(coro), repr(coro)
    super().__init__(loop=loop)
    if self._source_traceback:
        del self._source_traceback[-1]
    # 协程的迭代器
    self._coro = iter(coro)  # Use the iterator just in case.
    # 等待执行的 future
    self._fut_waiter = None
    # 如果为 True 任务会被取消
    self._must_cancel = False
    # 尽快调用实例的 _step 方法
    self._loop.call_soon(self._step)
    # 添加一个 task 到类 task 列表属性中
    self.__class__._all_tasks.add(self)
```
### 取消 task
```python
def cancel(self):
    # 如果任务已经完成返回 False
    if self.done():
        return False
    # 如果存在等待不为空
    if self._fut_waiter is not None:
		# 取消父类
        if self._fut_waiter.cancel():
            return True
    # It must be the case that self._step is already scheduled.
	# 取消状态设置为True,下次执行 _step 方法会把 task 取消掉
    self._must_cancel = True
    return True
```
### 步进执行 task
```python
def _step(self, value=None, exc=None):
	# 断言任务没有完成
	assert not self.done(), \
		'_step(): already done: {!r}, {!r}, {!r}'.format(self, value, exc)
	# 如果需要取消任务
	if self._must_cancel:
		# 设置取消异常
		if not isinstance(exc, futures.CancelledError):
			exc = futures.CancelledError()
		self._must_cancel = False
	# task 中的协程
	coro = self._coro
	# 等待为空
	self._fut_waiter = None
	# 设置当前 loop 正在执行的 task 为自己
	self.__class__._current_tasks[self._loop] = self
	try:
		# 存在异常 发送异常
		if exc is not None:
			result = coro.throw(exc)
		# 上一个协程的结果
		elif value is not None:
			result = coro.send(value)
		# 协程的下一步
		else:
			result = next(coro)
	except StopIteration as exc:
		self.set_result(exc.value)
	except futures.CancelledError as exc:
		super().cancel()  # I.e., Future.cancel(self).
	except Exception as exc:
		self.set_exception(exc)
	except BaseException as exc:
		self.set_exception(exc)
		raise
	else:
		if isinstance(result, futures.Future):
			# Yielded Future must come from Future.__iter__().
			if result._blocking:
				result._blocking = False
				result.add_done_callback(self._wakeup)
				self._fut_waiter = result
				if self._must_cancel:
					if self._fut_waiter.cancel():
						self._must_cancel = False
			else:
				self._loop.call_soon(
					self._step, None,
					RuntimeError(
						'yield was used instead of yield from '
						'in task {!r} with {!r}'.format(self, result)))
		elif result is None:
			# Bare yield relinquishes control for one event loop iteration.
			self._loop.call_soon(self._step)
		elif inspect.isgenerator(result):
			# Yielding a generator is just wrong.
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'yield was used instead of yield from for '
					'generator in task {!r} with {}'.format(
						self, result)))
		else:
			# Yielding something else is an error.
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'Task got bad yield: {!r}'.format(result)))
	finally:
		self.__class__._current_tasks.pop(self._loop)
		self = None  # Needed to break cycles when an exception occurs.
```
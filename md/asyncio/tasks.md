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
	# 迭代完成设置 task 结果，任务结束
	except StopIteration as exc:
		self.set_result(exc.value)
	# 协程被取消，取消任务
	except futures.CancelledError as exc:
		super().cancel()  # I.e., Future.cancel(self).
	# 出现异常，设置 task 的异常结果
	except Exception as exc:
		self.set_exception(exc)
	# 其他异常，设置异常结果并抛出异常
	except BaseException as exc:
		self.set_exception(exc)
		raise
	else:
		# 获取的结果是一个 future
		if isinstance(result, futures.Future):
			# Yielded Future must come from Future.__iter__().
			# 必须是通过 Future.__iter__() 方法返回的 future
			if result._blocking:
				# 阻塞设置为 False
				result._blocking = False
				# 当 future 完成时唤醒 task 继续执行
				result.add_done_callback(self._wakeup)
				# 设置 task 等待完成的 future 为当前 future
				self._fut_waiter = result
				# 如果任务被设置为取消
				if self._must_cancel:
					# 等待完成的 future 设置为取消
					if self._fut_waiter.cancel():
						# 任务已取消，重置 task 取消属性状态
						self._must_cancel = False
			# 下一次执行 _step 时设置异常
			else:
				self._loop.call_soon(
					self._step, None,
					RuntimeError(
						'yield was used instead of yield from '
						'in task {!r} with {!r}'.format(self, result)))
		# 协程任务没有返回结果，尽快执行下一次循环
		elif result is None:
			# Bare yield relinquishes control for one event loop iteration.
			self._loop.call_soon(self._step)
		# 如果协程返回了一个生成器
		elif inspect.isgenerator(result):
			# Yielding a generator is just wrong.
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'yield was used instead of yield from for '
					'generator in task {!r} with {}'.format(
						self, result)))
		# 返回其他的数据是错误的
		else:
			# Yielding something else is an error.
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'Task got bad yield: {!r}'.format(result)))
	# 最终：把 loop 当前执行的 task 从字典中删除
	finally:
		self.__class__._current_tasks.pop(self._loop)
		self = None  # Needed to break cycles when an exception occurs.
```
### 唤醒自己
```python
def _wakeup(self, future):
	try:
		value = future.result()
	except Exception as exc:
		# This may also be a cancellation.
		self._step(None, exc)
	else:
		self._step(value, None)
	self = None
```
## 判断对象是 future 或者 coroutine
```python
def async(coro_or_future, *, loop=None):
    # 如果参数是 future 并且传递的 loop 参数和 future 的 loop 相同直接返回
    if isinstance(coro_or_future, futures.Future):
        if loop is not None and loop is not coro_or_future._loop:
            raise ValueError('loop argument must agree with Future')
        return coro_or_future
	# 如果传递的参数是一个协程
    elif coroutines.iscoroutine(coro_or_future):
		# 设置 loop
        if loop is None:
            loop = events.get_event_loop()
		# 把协程包装成一个 task 并返回
        task = loop.create_task(coro_or_future)
        if task._source_traceback:
            del task._source_traceback[-1]
        return task
    else:
        raise TypeError('A Future or coroutine is required')
```
## 等待多个future或者coroutine完成
```python
@coroutine
def wait(fs, *, loop=None, timeout=None, return_when=ALL_COMPLETED):
    # fs 参数必须是一个 future 或者 coroutine 列表
    if isinstance(fs, futures.Future) or coroutines.iscoroutine(fs):
        raise TypeError("expect a list of futures, not %s" % type(fs).__name__)
    if not fs:
        raise ValueError('Set of coroutines/Futures is empty.')
	# 返回条件必须是三者之一
    if return_when not in (FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED):
        raise ValueError('Invalid return_when value: {}'.format(return_when))
	# 设置 loop
    if loop is None:
        loop = events.get_event_loop()
	# 判断列表中的对象是否满足条件
    fs = {async(f, loop=loop) for f in set(fs)}
	# 等待
    return (yield from _wait(fs, timeout, return_when, loop))
```
## def _wait()
```python
@coroutine
def _wait(fs, timeout, return_when, loop):
    assert fs, 'Set of Futures is empty.'
	# 创建一个新的 future 对象，用来判断任务是否完成
    waiter = futures.Future(loop=loop)
	# 超时对象
    timeout_handle = None
	# 如果 timeout 为真，设置一个超时对象，在这个时间内任务还没有完成，则抛出超时异常
    if timeout is not None:
        timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
	# 等待完成的任务数量
    counter = len(fs)

    def _on_completion(f):
		# 使用函数的变量
        nonlocal counter
		# 未完成的任务数量减一
        counter -= 1
		# 如果都完成了或者满足返回条件了
        if (counter <= 0 or
            return_when == FIRST_COMPLETED or
            return_when == FIRST_EXCEPTION and (not f.cancelled() and
                                                f.exception() is not None)):
			# 如果超时对象不为空，取消超时对象的执行
            if timeout_handle is not None:
                timeout_handle.cancel()
			# 如果等待的任务没有完成，设置结果。完成
            if not waiter.done():
                waiter.set_result(None)
	# 给每一个 future 对像添加完成回调，用于判断什么时候返回结果
    for f in fs:
        f.add_done_callback(_on_completion)
	# 返回它自己(Future.__iter__())给上层的 task，task 会等待这个 future 结束再执行下面的代码。而这个 future 结束的条件就在 上面的 _on_completion() 方法中。
    try:
        yield from waiter
    finally:
        if timeout_handle is not None:
            timeout_handle.cancel()
	# 满足返回条件后已经完成的和未完成的任务集合
    done, pending = set(), set()
    for f in fs:
        f.remove_done_callback(_on_completion)
        if f.done():
            done.add(f)
        else:
            pending.add(f)
    return done, pending
```
## 等待 future 执行结束
```python
@coroutine
def wait_for(fut, timeout, *, loop=None):
    if loop is None:
        loop = events.get_event_loop()
	# 如果超时参数为空，直接等待 future 结束
    if timeout is None:
        return (yield from fut)
	# 新建一个 future
    waiter = futures.Future(loop=loop)
	# 超时对象
    timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
	# 回调函数
    cb = functools.partial(_release_waiter, waiter)
	# fut参数必须是一个 future 或者 coroutine
    fut = async(fut, loop=loop)
	# 等 future 完成后调用回调函数
    fut.add_done_callback(cb)

    try:
        # 等待直到 future 完成或者超时
        try:
            yield from waiter
		# 如果任务被取消，future 也要被取消
        except futures.CancelledError:
            fut.remove_done_callback(cb)
            fut.cancel()
            raise

        if fut.done():
            return fut.result()
        else:
            fut.remove_done_callback(cb)
            fut.cancel()
            raise futures.TimeoutError()
    finally:
        timeout_handle.cancel()
```
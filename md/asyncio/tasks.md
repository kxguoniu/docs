# asyncio 之 tasks.py
## 摘要
在`asyncio`中每一个协程都会被包装成一个`task`。协程作为一个可迭代对象保存在`task`的属性中。
而`task`会借助`loop`无限循环自己的`_setup`方法。每一次`_setup`的执行都是对协程的一次迭代，如果协程的迭代结果是一个`future`，`_setup`会等待`future`完成后继续执行。当协程迭代结束之后`_setup`会把协程返回的结果放在`task`属性中，并结束无限循环。
## Task
### 类变量
实例对象的默认属性
```python
# 包含所有 task 的弱引用集合
_all_tasks = weakref.WeakSet()

# loop-task 字典，记录 loop 正在执行的 task
_current_tasks = {}

# task 在挂起状态时被销毁是否要记录日志
_log_destroy_pending = True
```
### 类方法
类方法有两个，一个是获取 loop 正在执行的 task，另一个则是获取 loop 所有的 task 集合
```python
@classmethod
def current_task(cls, loop=None):
    if loop is None:
        loop = events.get_event_loop()
    return cls._current_tasks.get(loop)

@classmethod
def all_tasks(cls, loop=None):
    if loop is None:
        loop = events.get_event_loop()
    return {t for t in cls._all_tasks if t._loop is loop}
```
### 初始化
```python
def __init__(self, coro, *, loop=None):
	# 初始化参数必须是一个协程对象
    assert coroutines.iscoroutine(coro), repr(coro)
    super().__init__(loop=loop)
    if self._source_traceback:
        del self._source_traceback[-1]
    # 把协程包装成一个迭代器
    self._coro = iter(coro)  # Use the iterator just in case.
    # task 被停止后等待完成的 future 对象
    self._fut_waiter = None
    # 如果中途取消 task 会把这个属性设置为 True
    self._must_cancel = False
    # 把 task 对象的方法添加到 loop 的待执行队列中
    self._loop.call_soon(self._step)
    # 把当前 task 添加到类的 tasks 集合中
    self.__class__._all_tasks.add(self)
```
### 取消 task
```python
def cancel(self):
    if self.done():
        return False
    # 如果 task 正在等待其他 future 的完成，取消 future
    if self._fut_waiter is not None:
        if self._fut_waiter.cancel():
            return True
	# 如果 task 没有等待的 future 说明 task 当前正在执行，把取消状态设置为 True。执行过程会取消掉 task
    self._must_cancel = True
    return True
```
### 步进执行 task
```python
def _step(self, value=None, exc=None):
	assert not self.done(), \
		'_step(): already done: {!r}, {!r}, {!r}'.format(self, value, exc)
	# 如果取消 task 属性为 True
	if self._must_cancel:
		if not isinstance(exc, futures.CancelledError):
			exc = futures.CancelledError()
		self._must_cancel = False
	# task 中的协程对象
	coro = self._coro
	self._fut_waiter = None
	# 设置自己为 loop 正在执行的 task
	self.__class__._current_tasks[self._loop] = self
	try:
		if exc is not None:
			result = coro.throw(exc)
		# 发送上次迭代的结果并获取这次迭代结果
		elif value is not None:
			result = coro.send(value)
		# 迭代一次协程
		else:
			result = next(coro)
	# 协程迭代结束，把停止异常的值(就是协程 return 的对象)设置为 task 的结果
	except StopIteration as exc:
		self.set_result(exc.value)
	# 携程被取消，task 也要跟着取消
	except futures.CancelledError as exc:
		super().cancel()  # I.e., Future.cancel(self).
	# 协程执行中出现异常，把异常信息记录在 task 中
	except Exception as exc:
		self.set_exception(exc)
	# 如果是其他异常，记录异常信息并抛出异常
	except BaseException as exc:
		self.set_exception(exc)
		raise
	else:
		# 迭代的结果是一个 future
		if isinstance(result, futures.Future):
			if result._blocking:
				result._blocking = False
				# 暂停 task 等到 future 完成之后再唤醒 task 继续执行
				result.add_done_callback(self._wakeup)
				# 设置 task 等待完成的 future 对象
				self._fut_waiter = result
				# 如果 task 被设置为取消
				if self._must_cancel:
					if self._fut_waiter.cancel():
						self._must_cancel = False
			else:
				self._loop.call_soon(
					self._step, None,
					RuntimeError(
						'yield was used instead of yield from '
						'in task {!r} with {!r}'.format(self, result)))
		# 迭代的结果为 None，放弃控制，执行下一次迭代
		elif result is None:
			# Bare yield relinquishes control for one event loop iteration.
			self._loop.call_soon(self._step)
		# 如果迭代结果是一个生成器，抛出异常
		elif inspect.isgenerator(result):
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'yield was used instead of yield from for '
					'generator in task {!r} with {}'.format(
						self, result)))
		# 超出意料的结果，抛出异常
		else:
			self._loop.call_soon(
				self._step, None,
				RuntimeError(
					'Task got bad yield: {!r}'.format(result)))
	finally:
		self.__class__._current_tasks.pop(self._loop)
		self = None  # Needed to break cycles when an exception occurs.
```
### 唤醒自己
task 等待的 future 已经完成，调用这个方法，让暂停的 task 继续执行下去。
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
## def async
如果参数是 future 直接返回，如果是协程则包装成 task 返回
```python
def async(coro_or_future, *, loop=None):
    # 如果参数是当前 loop 的 future 对象，直接返回
    if isinstance(coro_or_future, futures.Future):
        if loop is not None and loop is not coro_or_future._loop:
            raise ValueError('loop argument must agree with Future')
        return coro_or_future
	# 如果传递的参数是一个协程
    elif coroutines.iscoroutine(coro_or_future):
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
## def wait
该方法参数是一个列表，直到满足特定条件就返回
条件有三种
1. FIRST_COMPLETED	第一个对象完成就返回
2. FIRST_EXCEPTION  第一次出现异常就返回
3. ALL_COMPLETED	所有对象完成后返回

```python
@coroutine
def wait(fs, *, loop=None, timeout=None, return_when=ALL_COMPLETED):
    # fs 参数必须是一个 future 或者 coroutine 列表
    if isinstance(fs, futures.Future) or coroutines.iscoroutine(fs):
        raise TypeError("expect a list of futures, not %s" % type(fs).__name__)
    if not fs:
        raise ValueError('Set of coroutines/Futures is empty.')
    if return_when not in (FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED):
        raise ValueError('Invalid return_when value: {}'.format(return_when))
    if loop is None:
        loop = events.get_event_loop()
	# 把协程包装成 task
    fs = {async(f, loop=loop) for f in set(fs)}
    return (yield from _wait(fs, timeout, return_when, loop))
```
### def _wait()
```python
@coroutine
def _wait(fs, timeout, return_when, loop):
    assert fs, 'Set of Futures is empty.'
	# 创建一个新的 future 对象，用来判断任务是否完成
    waiter = futures.Future(loop=loop)
    timeout_handle = None
    if timeout is not None:
		# 超时之后好没有完成就把 waiter 的结果设置为 None
        timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
	# 任务计数器
    counter = len(fs)

    def _on_completion(f):
        nonlocal counter
		# 计数器减一
        counter -= 1
		# 如果满足了设置的条件
        if (counter <= 0 or
            return_when == FIRST_COMPLETED or
            return_when == FIRST_EXCEPTION and (not f.cancelled() and
                                                f.exception() is not None)):
            # 取消超时回调对象
			if timeout_handle is not None:
                timeout_handle.cancel()
			# 如果 waiter 没有完成，设置结果，更新状态
            if not waiter.done():
                waiter.set_result(None)
	# 给 fs 集合里的每一个对象添加回调函数
    for f in fs:
        f.add_done_callback(_on_completion)
	# 阻塞直到满足条件后，waiter 被设置结果。
    try:
        yield from waiter
    finally:
        if timeout_handle is not None:
            timeout_handle.cancel()
	# 返回完成的集合和未完成的集合，并删除回调函数
    done, pending = set(), set()
    for f in fs:
        f.remove_done_callback(_on_completion)
        if f.done():
            done.add(f)
        else:
            pending.add(f)
    return done, pending
```
## def wait_for
这个方法主要是给 future 添加一个超时等待
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
	# 超时之后还没有完成直接设置 waiter 结果为 None
    timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
	# 回调函数
    cb = functools.partial(_release_waiter, waiter)
    fut = async(fut, loop=loop)
	# 给 future 设置回调函数
    fut.add_done_callback(cb)

    try:
        # 等待 future 完成直到超时
        try:
            yield from waiter
		# 如果 waiter 被取消， future 也应该被取消
        except futures.CancelledError:
            fut.remove_done_callback(cb)
            fut.cancel()
            raise
		# 如果 future 完成了，则返回结果
        if fut.done():
            return fut.result()
		# 这种是由于超时退出的，抛出超时异常
        else:
            fut.remove_done_callback(cb)
            fut.cancel()
            raise futures.TimeoutError()
    finally:
        timeout_handle.cancel()
```
## def as_completed
```python
def as_completed(fs, *, loop=None, timeout=None):
	"""
        for f in as_completed(fs):
            result = yield from f  # The 'yield from' may raise.
            # Use result.
    """
    if isinstance(fs, futures.Future) or coroutines.iscoroutine(fs):
        raise TypeError("expect a list of futures, not %s" % type(fs).__name__)
    loop = loop if loop is not None else events.get_event_loop()
	# 任务集合
    todo = {async(f, loop=loop) for f in set(fs)}
    from .queues import Queue
	# 结果队列
    done = Queue(loop=loop)
    timeout_handle = None

	# 超时之后把所有未完成的任务清空，并向结果队列添加相应数量的 None 值
    def _on_timeout():
        for f in todo:
            f.remove_done_callback(_on_completion)
            done.put_nowait(None)  # Queue a dummy value for _wait_for_one().
        todo.clear()  # Can't do todo.remove(f) in the loop.

	# 完成之后把 future 从待完成集合中删除并把它添加到结果队列中
    def _on_completion(f):
        if not todo:
            return  # _on_timeout() was here first.
        todo.remove(f)
        done.put_nowait(f)
		# 如果任务在超时之前全部完成，取消超时回调函数
        if not todo and timeout_handle is not None:
            timeout_handle.cancel()

	# 这个协程从结果队列中取出完成的任务病返回其结果
    @coroutine
    def _wait_for_one():
        f = yield from done.get()
        if f is None:
            # Dummy value from _on_timeout().
            raise futures.TimeoutError
        return f.result()  # May raise f.exception().

	# 为所有的 future 添加回调函数
    for f in todo:
        f.add_done_callback(_on_completion)
	# 设置超时回调函数
    if todo and timeout is not None:
        timeout_handle = loop.call_later(timeout, _on_timeout)
	# 迭代时返回一个协程
    for _ in range(len(todo)):
        yield _wait_for_one()
```
## def sleep
```python
@coroutine
def sleep(delay, result=None, *, loop=None):
    # 创建一个 future 对象
    future = futures.Future(loop=loop)
	# delay 秒之后调用函数设置 future 的结果为 result 参数
    h = future._loop.call_later(delay,
                                future._set_result_unless_cancelled, result)
	# 等待 future 的完成
    try:
        return (yield from future)
    finally:
        h.cancel()
```
## def gather
gather 是把所有的 future 或者 coroutine 包装在一个 future 里面并把它返回.这个future的结果是一个列表,里面按照顺序排列传递的future 或者 coroutine 结果
```python
def gather(*coros_or_futures, loop=None, return_exceptions=False):
	# 如果没有给参数,返回一个结果为空列表的 future
    if not coros_or_futures:
        outer = futures.Future(loop=loop)
        outer.set_result([])
        return outer
	# 参数-future 字典
    arg_to_fut = {}
    for arg in set(coros_or_futures):
		# 把协程包装成 task
        if not isinstance(arg, futures.Future):
            fut = async(arg, loop=loop)
            if loop is None:
                loop = fut._loop
            # The caller cannot control this future, the "destroy pending task"
            # warning should not be emitted.
            fut._log_destroy_pending = False
        else:
            fut = arg
            if loop is None:
                loop = fut._loop
            elif fut._loop is not loop:
                raise ValueError("futures are tied to different event loops")
        arg_to_fut[arg] = fut
	# 子任务列表
    children = [arg_to_fut[arg] for arg in coros_or_futures]
    nchildren = len(children)
	# 把所有的子任务包装在一个特定的 future 中
    outer = _GatheringFuture(children, loop=loop)
    nfinished = 0
	# 子任务结果列表
    results = [None] * nchildren

	# 子任务的回调函数
    def _done_callback(i, fut):
        nonlocal nfinished
		# 如果 outer 已经完成，这种情况是 outer 设置了异常。不做任何操作
        if outer.done():
            if not fut.cancelled():
                # Mark exception retrieved.
                fut.exception()
            return
		# 如果子任务被取消了，
        if fut.cancelled():
            res = futures.CancelledError()
            if not return_exceptions:
                outer.set_exception(res)
                return
		# 如果子任务出现了异常
        elif fut._exception is not None:
            res = fut.exception()  # Mark exception retrieved.
            if not return_exceptions:
                outer.set_exception(res)
                return
        else:
            res = fut._result
		# 设置第 i 个子任务的结果
        results[i] = res
        nfinished += 1
		# 如果子任务已经全部完成，设置父任务的结果列表
        if nfinished == nchildren:
            outer.set_result(results)

	# 给所有的子任务设置回调函数
    for i, fut in enumerate(children):
        fut.add_done_callback(functools.partial(_done_callback, i))
    return outer
```
## def shield
```python
def shield(arg, *, loop=None):
    inner = async(arg, loop=loop)
    if inner.done():
        # Shortcut.
        return inner
    loop = inner._loop
    outer = futures.Future(loop=loop)

    def _done_callback(inner):
        if outer.cancelled():
            if not inner.cancelled():
                # Mark inner's result as retrieved.
                inner.exception()
            return

        if inner.cancelled():
            outer.cancel()
        else:
            exc = inner.exception()
            if exc is not None:
                outer.set_exception(exc)
            else:
                outer.set_result(inner.result())

    inner.add_done_callback(_done_callback)
    return outer
```
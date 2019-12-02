# asyncio 之 tasks.py
[toc]
## 摘要
在`asyncio`中每一个协程都会被包装成一个`task`。协程作为一个可迭代对象保存在`task`的属性中。
而`task`会借助`loop`无限循环自己的`_setup`方法。每一次`_setup`的执行都是对协程的一次迭代，如果协程的迭代结果是一个`future`，`_setup`会等待`future`完成后继续执行。当协程迭代结束之后`_setup`会把协程返回的结果放在`task`属性中，并结束无限循环。
## class Task
### 类属性
如果设置为`False`，当`task`被销毁时如果`task`状态为`pending`则不会记录日志
```python
_log_destroy_pending = True
```
### 类方法
类方法有两个，一个是获取 loop 正在执行的 task，另一个则是获取 loop 所有的 task 集合
```python
@classmethod
def current_task(cls, loop=None):
	warnings.warn("Task.current_task() is deprecated, "
				  "use asyncio.current_task() instead",
				  PendingDeprecationWarning,
				  stacklevel=2)
	if loop is None:
		loop = events.get_event_loop()
	return current_task(loop)

@classmethod
def all_tasks(cls, loop=None):
	warnings.warn("Task.all_tasks() is deprecated, "
				  "use asyncio.all_tasks() instead",
				  PendingDeprecationWarning,
				  stacklevel=2)
	return _all_tasks_compat(loop)
```
### 初始化
```python
def __init__(self, coro, *, loop=None):
	super().__init__(loop=loop)
	if self._source_traceback:
		del self._source_traceback[-1]
	# 如果参数不是协程，抛出异常
	if not coroutines.iscoroutine(coro):
		self._log_destroy_pending = False
		raise TypeError(f"a coroutine was expected, got {coro!r}")

	# 如果中途取消 task 会把这个属性设置为 True,task的取消不是同步的
	self._must_cancel = False
	# task 被停止后等待完成的 future 对象
	self._fut_waiter = None
	self._coro = coro
	self._context = contextvars.copy_context()
	# 把 task 对象的方法添加到 loop 的待执行队列中
	self._loop.call_soon(self.__step, context=self._context)
	# 注册task，添加到task集合中
	_register_task(self)
```
### 删除 task
```python
def __del__(self):
	if self._state == futures._PENDING and self._log_destroy_pending:
		context = {
			'task': self,
			'message': 'Task was destroyed but it is pending!',
		}
		if self._source_traceback:
			context['source_traceback'] = self._source_traceback
		self._loop.call_exception_handler(context)
	super().__del__()
```
### 设置结果/异常
当前版本的`asyncio`增加了限制，`task`已经不支持这两个方法了
```python
def set_result(self, result):
	raise RuntimeError('Task does not support set_result operation')

def set_exception(self, exception):
	raise RuntimeError('Task does not support set_exception operation')
```
### 获取task的堆栈列表
```python
    def get_stack(self, *, limit=None):
        return base_tasks._task_get_stack(self, limit)
```
### 打印task的堆栈或者回溯信息
```python
def print_stack(self, *, limit=None, file=None):
	return base_tasks._task_print_stack(self, limit, file)
```
### 取消 task
```python
def cancel(self):
	self._log_traceback = False
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
def __step(self, exc=None):
    if self.done():
        raise futures.InvalidStateError(
            f'_step(): already done: {self!r}, {exc!r}')
    # 如果需要取消task，设置异常为取消异常
    if self._must_cancel:
        if not isinstance(exc, futures.CancelledError):
            exc = futures.CancelledError()
        self._must_cancel = False
    # task 的协程对象
    coro = self._coro
    self._fut_waiter = None
    # 设置loop当前执行的task为自己
    _enter_task(self._loop, self)
    # 执行一次协程
    try:
        if exc is None:
            result = coro.send(None)
        else:
            result = coro.throw(exc)
    # 协程执行完成，设置结果
    except StopIteration as exc:
        # 如果task被取消，设置异常
        if self._must_cancel:
            self._must_cancel = False
            super().set_exception(futures.CancelledError())
        # 设置协程的结果
        else:
            super().set_result(exc.value)
    # 如果出现取消异常说明task被取消了，具体取消在这里执行
    except futures.CancelledError:
        super().cancel()  # I.e., Future.cancel(self).
    # 如果出现通用异常，设置异常结果
    except Exception as exc:
        super().set_exception(exc)
    # 如果抛出基础异常，向上层传递异常
    except BaseException as exc:
        super().set_exception(exc)
        raise
    else:
        blocking = getattr(result, '_asyncio_future_blocking', None)
        # 返回的result是一个 future 对象
        if blocking is not None:
            # future对象的loop必须是当前task的loop
            if futures._get_loop(result) is not self._loop:
                new_exc = RuntimeError(
                    f'Task {self!r} got Future '
                    f'{result!r} attached to a different loop')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)
            elif blocking:
                # 返回的future对象不能是自己
                if result is self:
                    new_exc = RuntimeError(
                        f'Task cannot await on itself: {self!r}')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)
                else:
                    result._asyncio_future_blocking = False
                    # 添加回调函数，等future完成之后再通知task继续执行
                    result.add_done_callback(
                        self.__wakeup, context=self._context)
                    # 把阻塞task的future对象设置为 当前future
                    self._fut_waiter = result
                    # 如果需要取消task
                    if self._must_cancel:
                        if self._fut_waiter.cancel():
                            self._must_cancel = False
            else:
                new_exc = RuntimeError(
                    f'yield was used instead of yield from '
                    f'in task {self!r} with {result!r}')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)

        elif result is None:
            self._loop.call_soon(self.__step, context=self._context)
        # 协程返回一个生成器或者返回其他结果都是错误的
        elif inspect.isgenerator(result):
            new_exc = RuntimeError(
                f'yield was used instead of yield from for '
                f'generator in task {self!r} with {result!r}')
            self._loop.call_soon(
                self.__step, new_exc, context=self._context)
        else:
            new_exc = RuntimeError(f'Task got bad yield: {result!r}')
            self._loop.call_soon(
                self.__step, new_exc, context=self._context)
    # 把loop当前的task删除
    finally:
        _leave_task(self._loop, self)
        self = None  # Needed to break cycles when an exception occurs.
```
### 唤醒自己
task 等待的 future 已经完成，调用这个方法，让暂停的 task 继续执行下去。
```python
def __wakeup(self, future):
	try:
		future.result()
	except Exception as exc:
		# This may also be a cancellation.
		self.__step(exc)
	else:
		self.__step()
	self = None  # Needed to break cycles when an exception occurs.
```
## 文件内方法
### 创建task
```python
def create_task(coro):
    loop = events.get_running_loop()
    return loop.create_task(coro)
```
### def wait
该方法参数是一个列表，直到满足特定条件就返回
```python
FIRST_COMPLETED = concurrent.futures.FIRST_COMPLETED	# 只要有一个完成就返回
FIRST_EXCEPTION = concurrent.futures.FIRST_EXCEPTION	# 只要有一个出现异常就返回
ALL_COMPLETED = concurrent.futures.ALL_COMPLETED		# 所有的都完成才返回

async def wait(fs, *, loop=None, timeout=None, return_when=ALL_COMPLETED):
    # fs参数必须是一个协程或者future列表
	if futures.isfuture(fs) or coroutines.iscoroutine(fs):
        raise TypeError(f"expect a list of futures, not {type(fs).__name__}")
    if not fs:
        raise ValueError('Set of coroutines/Futures is empty.')
    if return_when not in (FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED):
        raise ValueError(f'Invalid return_when value: {return_when}')
    if loop is None:
        loop = events.get_event_loop()
	# 把列表中的协程包装task
    fs = {ensure_future(f, loop=loop) for f in set(fs)}
    return await _wait(fs, timeout, return_when, loop)
```
#### def _wait
```python
async def _wait(fs, timeout, return_when, loop):
    assert fs, 'Set of Futures is empty.'
	# 创建一个新的future来判断任务是否已经完成
    waiter = loop.create_future()
	# 如果超时参数不为None，设置超时回调函数
    timeout_handle = None
    if timeout is not None:
        timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
    # 任务计数器
	counter = len(fs)

    def _on_completion(f):
        nonlocal counter
		# 任务完成，计数器减一
        counter -= 1
		# 满足条件
        if (counter <= 0 or
            return_when == FIRST_COMPLETED or
            return_when == FIRST_EXCEPTION and (not f.cancelled() and
                                                f.exception() is not None)):
            if timeout_handle is not None:
                timeout_handle.cancel()
            if not waiter.done():
                waiter.set_result(None)
	# 为每一个future添加回调函数
    for f in fs:
        f.add_done_callback(_on_completion)
	# 阻塞程序直到满足条件返回
    try:
        await waiter
    finally:
        if timeout_handle is not None:
            timeout_handle.cancel()
        for f in fs:
            f.remove_done_callback(_on_completion)
	# 已完成和未完成的任务集合
    done, pending = set(), set()
    for f in fs:
        if f.done():
            done.add(f)
        else:
            pending.add(f)
    return done, pending
```
### def ensure_future
确保参数是一个可用的`future`对象或者是一个`coroutine`对象
```python
def ensure_future(coro_or_future, *, loop=None):
	# 如果参数是协程对象，把协程包装成一个task并返回
    if coroutines.iscoroutine(coro_or_future):
        if loop is None:
            loop = events.get_event_loop()
        task = loop.create_task(coro_or_future)
        if task._source_traceback:
            del task._source_traceback[-1]
        return task
	# 如果参数是一个future对象直接返回
    elif futures.isfuture(coro_or_future):
        if loop is not None and loop is not futures._get_loop(coro_or_future):
            raise ValueError('loop argument must agree with Future')
        return coro_or_future
    elif inspect.isawaitable(coro_or_future):
        return ensure_future(_wrap_awaitable(coro_or_future), loop=loop)
    else:
        raise TypeError('An asyncio.Future, a coroutine or an awaitable is '
                        'required')

@coroutine
def _wrap_awaitable(awaitable):
    return (yield from awaitable.__await__())
```

### def _release_waiter
```python
def _release_waiter(waiter, *args):
    if not waiter.done():
        waiter.set_result(None)
```
### def wait_for
```python
async def wait_for(fut, timeout, *, loop=None):
    if loop is None:
        loop = events.get_event_loop()

    if timeout is None:
        return await fut

	# 直接取消future并抛出超时异常
    if timeout <= 0:
        fut = ensure_future(fut, loop=loop)
        if fut.done():
            return fut.result()
        fut.cancel()
        raise futures.TimeoutError()

    waiter = loop.create_future()
    timeout_handle = loop.call_later(timeout, _release_waiter, waiter)
    cb = functools.partial(_release_waiter, waiter)
	# 给future添加回调函数
    fut = ensure_future(fut, loop=loop)
    fut.add_done_callback(cb)

    try:
        try:
            await waiter
        except futures.CancelledError:
            fut.remove_done_callback(cb)
            fut.cancel()
            raise

        if fut.done():
            return fut.result()
        else:
            fut.remove_done_callback(cb)
            await _cancel_and_wait(fut, loop=loop)
            raise futures.TimeoutError()
    finally:
        timeout_handle.cancel()
```
### def sleep
```python
@types.coroutine
def __sleep0():
    yield

async def sleep(delay, result=None, *, loop=None):
    if delay <= 0:
        await __sleep0()
        return result

    if loop is None:
        loop = events.get_event_loop()
    future = loop.create_future()
    h = loop.call_later(delay,
                        futures._set_result_unless_cancelled,
                        future, result)
    try:
        return await future
    finally:
        h.cancel()
```
## def as_completed
```python
def as_completed(fs, *, loop=None, timeout=None):
    """
        for f in as_completed(fs):
            result = await f  # The 'await' may raise.
            # Use result.
    """
    if futures.isfuture(fs) or coroutines.iscoroutine(fs):
        raise TypeError(f"expect a list of futures, not {type(fs).__name__}")
    loop = loop if loop is not None else events.get_event_loop()
    # 任务集合
    todo = {ensure_future(f, loop=loop) for f in set(fs)}
    from .queues import Queue  # Import here to avoid circular import problem.
    # 结果队列
    done = Queue(loop=loop)
    timeout_handle = None

    # 超时之后把所有的未完成的任务清空，并向结果队列添加相应数量的None值
    def _on_timeout():
        for f in todo:
            f.remove_done_callback(_on_completion)
            done.put_nowait(None)  # Queue a dummy value for _wait_for_one().
        todo.clear()  # Can't do todo.remove(f) in the loop.

    # 完成之后把 future 从待完成集合中删除，并把它添加到结果队列中
    def _on_completion(f):
        if not todo:
            return  # _on_timeout() was here first.
        todo.remove(f)
        done.put_nowait(f)
        # 如果任务全部完成，取消超时回调函数
        if not todo and timeout_handle is not None:
            timeout_handle.cancel()

    # 这个协程从结果队列中取出完成的任务并返回其结果
    async def _wait_for_one():
        f = await done.get()
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
## def gather
gather 是把所有的 future 或者 coroutine 包装在一个 future 里面并把它返回.这个future的结果是一个列表,里面按照顺序排列传递的future 或者 coroutine 结果
```python
def gather(*coros_or_futures, loop=None, return_exceptions=False):
    # 如果参数为空，返回一个结果为空列表的future
    if not coros_or_futures:
        if loop is None:
            loop = events.get_event_loop()
        outer = loop.create_future()
        outer.set_result([])
        return outer

    def _done_callback(fut):
        # 任务完成的数量增加一
        nonlocal nfinished
        nfinished += 1
        # 如果 outer 已经完成，这种情况是outer设置了异常，不做任何操作
        if outer.done():
            # 检查异常
            if not fut.cancelled():
                fut.exception()
            return

        # 如果不需要异常作为结果返回，那么当出现异常的时候就给 outer 设置异常。
        if not return_exceptions:
            if fut.cancelled():
                exc = futures.CancelledError()
                outer.set_exception(exc)
                return
            else:
                exc = fut.exception()
                if exc is not None:
                    outer.set_exception(exc)
                    return
        # 所有的任务都已经被完成
        if nfinished == nfuts:
            results = []
            # 循环子任务列表，取出完成任务的结果
            for fut in children:
                if fut.cancelled():
                    res = futures.CancelledError()
                else:
                    res = fut.exception()
                    if res is None:
                        res = fut.result()
                results.append(res)
            # 如果取消的标识为真，设置取消异常
            if outer._cancel_requested:
                outer.set_exception(futures.CancelledError())
            # 设置结果
            else:
                outer.set_result(results)
    # 参数-future 字典
    arg_to_fut = {}
    children = []
    nfuts = 0
    nfinished = 0
    for arg in coros_or_futures:
        if arg not in arg_to_fut:
            # 把参数中的协程包装成字典
            fut = ensure_future(arg, loop=loop)
            if loop is None:
                loop = futures._get_loop(fut)
            # 参数不是一个future，关闭删除未完成的task的警告
            if fut is not arg:
                fut._log_destroy_pending = False
            # 计数器增加一
            nfuts += 1
            arg_to_fut[arg] = fut
            # 添加回调函数
            fut.add_done_callback(_done_callback)
        else:
            fut = arg_to_fut[arg]
        # 子任务列表添加一个 future
        children.append(fut)
    # 把所有的子任务放在一个特定的 future 中
    outer = _GatheringFuture(children, loop=loop)
    return outer
```
## def shield
```python
def shield(arg, *, loop=None):
    inner = ensure_future(arg, loop=loop)
    if inner.done():
        # Shortcut.
        return inner
    loop = futures._get_loop(inner)
    outer = loop.create_future()

    def _inner_done_callback(inner):
        # 如果 outer 被取消，返回什么也不做
        if outer.cancelled():
            if not inner.cancelled():
                # Mark inner's result as retrieved.
                inner.exception()
            return
        # 把inter的状态信息复制到outer上
        if inner.cancelled():
            outer.cancel()
        else:
            exc = inner.exception()
            if exc is not None:
                outer.set_exception(exc)
            else:
                outer.set_result(inner.result())

    # 如果outer被取消，删除inter的回调函数
    def _outer_done_callback(outer):
        if not inner.done():
            inner.remove_done_callback(_inner_done_callback)
    # 添加回调函数
    inner.add_done_callback(_inner_done_callback)
    outer.add_done_callback(_outer_done_callback)
    return outer
```
### def run_coroutine_threadsafe
将一个协程交给给定的loop执行，返回一个 concurrent.futures.Future 对象用于访问结果。
```python
def run_coroutine_threadsafe(coro, loop):
    if not coroutines.iscoroutine(coro):
        raise TypeError('A coroutine object is required')
	# 并发库的future对象
    future = concurrent.futures.Future()

    def callback():
        try:
            futures._chain_future(ensure_future(coro, loop=loop), future)
        except Exception as exc:
            if future.set_running_or_notify_cancel():
                future.set_exception(exc)
            raise

    loop.call_soon_threadsafe(callback)
    return future
```
## 文件内置方法
```python
# 所有loop的task对象都在这个弱引用集合里面
_all_tasks = weakref.WeakSet()
# loop与其正在执行的task的字典
_current_tasks = {}

# 注册 task
def _register_task(task):
    _all_tasks.add(task)

# 进入task
def _enter_task(loop, task):
    current_task = _current_tasks.get(loop)
    if current_task is not None:
        raise RuntimeError(f"Cannot enter into task {task!r} while another "
                           f"task {current_task!r} is being executed.")
    _current_tasks[loop] = task

# 退出task
def _leave_task(loop, task):
    current_task = _current_tasks.get(loop)
    if current_task is not task:
        raise RuntimeError(f"Leaving task {task!r} does not match "
                           f"the current task {current_task!r}.")
    del _current_tasks[loop]

# 取消task的注册
def _unregister_task(task):
    _all_tasks.discard(task)
```
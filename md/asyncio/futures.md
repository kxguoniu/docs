#asyncio 之 futures.py
取消，获取结果，获取异常结果，完成回调，删除完成回调，设置结果，设置异常，包装 future
## TODO
```python
# Future 对象的三种状态
_PENDING = 'PENDING'
_CANCELLED = 'CANCELLED'
_FINISHED = 'FINISHED'

_PY34 = sys.version_info >= (3, 4)

Error = concurrent.futures._base.Error
CancelledError = concurrent.futures.CancelledError
TimeoutError = concurrent.futures.TimeoutError

STACK_DEBUG = logging.DEBUG - 1  # heavy-duty debugging
```
## Future 类
### 类变量
实例变量的默认值
```python
class Future:
	_state = _PENDING
    _result = None
    _exception = None
    _loop = None
    _source_traceback = None

    _blocking = False  # proper use of future (yield vs yield from)

    _log_traceback = False   # Used for Python 3.4 and later
    _tb_logger = None        # Used for Python 3.3 only
```
### 实例初始化
```python
def __init__(self, *, loop=None):
	# 如果没有 loop 获取默认的 loop
	if loop is None:
		self._loop = events.get_event_loop()
	else:
		self._loop = loop
	# future 取消或完成之后执行的可调用对象列表
	self._callbacks = []
	# 调试模式的代码回溯对象
	if self._loop.get_debug():
		self._source_traceback = traceback.extract_stack(sys._getframe(1))
```
### 取消 future
```python
def cancel(self):
	# 如果 future 已经取消或者执行完成 返回 False
	if self._state != _PENDING:
		return False
	# 把 future 状态设置为取消
	self._state = _CANCELLED
	# 调度等待 future 取消或者完成后执行的可调用对象列表
	self._schedule_callbacks()
	return True

def _schedule_callbacks(self):
	# 复制回调对象列表
	callbacks = self._callbacks[:]
	if not callbacks:
		return
	# 清空 future 的回调列表
	self._callbacks[:] = []
	# 尽快执行回调列表中的可调用对象
	for callback in callbacks:
		self._loop.call_soon(callback, self)
```
### 判断 future 的状态
```python
def cancelled(self):
	# 如果 future 已经取消返回 True
	return self._state == _CANCELLED

def done(self):
	# 如果 future 已经取消或完成返回 True
	return self._state != _PENDING
```
### 获取 future 结果
```python
    def result(self):
        # 如果 future 已经取消。抛出取消异常
        if self._state == _CANCELLED:
            raise CancelledError
		# 如果 future 还没有完成，抛出异常
        if self._state != _FINISHED:
            raise InvalidStateError('Result is not ready.')
		# 清理 future
        self._log_traceback = False
        if self._tb_logger is not None:
            self._tb_logger.clear()
            self._tb_logger = None
		# 如果执行任务过程中出现了异常，则异常信息会保存在future的异常属性中
		# 并在此时抛出异常
        if self._exception is not None:
            raise self._exception
		# 返回 future 结果
        return self._result
```
### 获取 future 的异常信息
future 没有被取消，并且任务已经完成。如果存在异常返回异常信息，否则返回 None。
```python
def exception(self):
	if self._state == _CANCELLED:
		raise CancelledError
	if self._state != _FINISHED:
		raise InvalidStateError('Exception is not set.')
	self._log_traceback = False
	if self._tb_logger is not None:
		self._tb_logger.clear()
		self._tb_logger = None
	return self._exception
```
### 添加回调对象
```python
def add_done_callback(self, fn):
	# 如果 future 已经取消或者完成，立即执行回调对象
	if self._state != _PENDING:
		self._loop.call_soon(fn, self)
	# 否则添加到 future 的回调对象列表中
	else:
		self._callbacks.append(fn)
```
### 删除回调对象
删除 future 回调列表属性中的回调对象并返回删除的数量
```python
def remove_done_callback(self, fn):
	filtered_callbacks = [f for f in self._callbacks if f != fn]
	removed_count = len(self._callbacks) - len(filtered_callbacks)
	if removed_count:
		self._callbacks[:] = filtered_callbacks
	return removed_count
```
### 设置 future 结果
```python
def set_result(self, result):
	# 如果 future 已经取消或者完成，抛出异常信息。
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	# 设置结果
	self._result = result
	# 更新 future 状态为已完成
	self._state = _FINISHED
	# 调度 future 回调列表中的可调用对象
	self._schedule_callbacks()
```
### 设置 future 异常
```python
def set_exception(self, exception):
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	# 如果传递的异常是一个类型，比如是一个异常类
	if isinstance(exception, type):
		exception = exception()
	# 设置异常
	self._exception = exception
	# 更新 future 状态为已完成
	self._state = _FINISHED
	# 调度回调列表
	self._schedule_callbacks()
	# 异常日志
	if _PY34:
		self._log_traceback = True
	else:
		self._tb_logger = _TracebackLogger(self, exception)
		# Arrange for the logger to be activated after all callbacks
		# have had a chance to call result() or exception().
		self._loop.call_soon(self._tb_logger.activate)
```
### 从其他的 future 复制状态
把目标 future 的状态以及结果复制到当前 future 上。
目标 future 可能是 concurrent.futures.Future
```python
    def _copy_state(self, other):
        assert other.done()
        if self.cancelled():
            return
        assert not self.done()
        if other.cancelled():
            self.cancel()
        else:
            exception = other.exception()
            if exception is not None:
                self.set_exception(exception)
            else:
                result = other.result()
                self.set_result(result)
```
### TODO
```python
def __iter__(self):
	if not self.done():
		self._blocking = True
		yield self  # This tells Task to wait for completion.
	assert self.done(), "yield from wasn't used with future"
	return self.result()  # May raise too.
```
## 包装 future
把 concurrent.futures.Future 对象包装成 asyncio 的 future
```python
def wrap_future(fut, *, loop=None):
	# 如果是 asyncio 的 future 对象直接返回
    if isinstance(fut, Future):
        return fut
	# 断言 必须是 concurrent.futures.Future 的实例
    assert isinstance(fut, concurrent.futures.Future), \
        'concurrent.futures.Future is expected, got {!r}'.format(fut)
    # 如果没有传递 loop，获取默认的 loop
	if loop is None:
        loop = events.get_event_loop()
	# 创建一个 future 对象
    new_future = Future(loop=loop)
	# 如果新的 future 对象取消，旧的 future 也要取消
    def _check_cancel_other(f):
        if f.cancelled():
            fut.cancel()
	# 给新的 future 添加一个回调对象
    new_future.add_done_callback(_check_cancel_other)
	# 旧的 future 添加一个回调对象，如果旧的 future 对象完成之后会把状态和结果复制到新的 future 对象中
    fut.add_done_callback(
        lambda future: loop.call_soon_threadsafe(
            new_future._copy_state, future))
	# 返回新的 future 对象
    return new_future
```
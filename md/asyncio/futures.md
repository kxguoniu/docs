# asyncio 之 futures.py
# head
文件主要实现了一个 Future 类和一个 wrap_future 方法
Future 类中主要的方法有 设置结果、设置异常、获取结果、获取异常、添加回调函数、删除回调函数、取消future等。
wrap_future 方法主要作用是把其他模块的 future 对象关联到当前模块的 future 对象上，实现 future 对象之间的状态和内容同步。
## 文件内常量
文件开头定义了三种 future 状态常量。分别是待执行、已取消、已完成。
三种异常分别是基础异常、取消异常、超时异常。
```python
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
实例初始化的时候的默认值
```python
class Future:
	_state = _PENDING
    _result = None
    _exception = None
    _loop = None
    _source_traceback = None

    _blocking = False

    _log_traceback = False   # >= 3.4
    _tb_logger = None        # = 3.3
```
### 实例初始化
future 实例初始化除了默认的类变量还有一个可调用对象列表的属性。这个列表中的对象会在 future 取消或者完成之后依次调用。
```python
def __init__(self, *, loop=None):
	if loop is None:
		self._loop = events.get_event_loop()
	else:
		self._loop = loop
	self._callbacks = []
	if self._loop.get_debug():
		self._source_traceback = traceback.extract_stack(sys._getframe(1))
```
### 取消 future
把 future 对象的状态设置为取消并依次执行回调函数列表中的对象。
```python
def cancel(self):
	if self._state != _PENDING:
		return False
	self._state = _CANCELLED
	# 执行回调对象
	self._schedule_callbacks()
	return True
# 把需要调用的对象依次放进 loop 循环中执行，并清空列表
def _schedule_callbacks(self):
	callbacks = self._callbacks[:]
	if not callbacks:
		return
	self._callbacks[:] = []
	for callback in callbacks:
		self._loop.call_soon(callback, self)
```
### 获取 future 的状态
获取 future 对象的状态方法有两个，一个获取取消状态另一个获取完成状态
```python
def cancelled(self):
	return self._state == _CANCELLED

def done(self):
	return self._state != _PENDING
```
### 获取 future 的结果
如果 future 关联的任务执行结束返回任务的结果，如果执行过程中出现了异常则抛出异常信息。
```python
def result(self):
	if self._state == _CANCELLED:
		raise CancelledError
	if self._state != _FINISHED:
		raise InvalidStateError('Result is not ready.')
	self._log_traceback = False
	if self._tb_logger is not None:
		self._tb_logger.clear()
		self._tb_logger = None
	# 如果执行任务过程中出现了异常，则异常信息会保存在future的异常属性中
	# 并在此时抛出异常
	if self._exception is not None:
		raise self._exception
	return self._result
```
### 获取 future 的异常信息
future 完成后获取产生的异常信息，这个与获取结果不同，获取结果是抛出异常信息。
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
给 future 添加回调对象，如果future已经完成则立刻执行。
```python
def add_done_callback(self, fn):
	if self._state != _PENDING:
		self._loop.call_soon(fn, self)
	else:
		self._callbacks.append(fn)
```
### 删除回调对象
删除 future 回调列表中的回调对象并返回删除的数量
```python
def remove_done_callback(self, fn):
	filtered_callbacks = [f for f in self._callbacks if f != fn]
	removed_count = len(self._callbacks) - len(filtered_callbacks)
	if removed_count:
		self._callbacks[:] = filtered_callbacks
	return removed_count
```
### 设置 future 结果
future 关联的任务已经完成，给future设置结果，更新状态，依次执行回调对象
```python
def set_result(self, result):
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	self._result = result
	self._state = _FINISHED
	self._schedule_callbacks()
```
### 设置 future 异常
同设置结果
```python
def set_exception(self, exception):
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	if isinstance(exception, type):
		exception = exception()
	self._exception = exception
	self._state = _FINISHED
	self._schedule_callbacks()
	if _PY34:
		self._log_traceback = True
	else:
		self._tb_logger = _TracebackLogger(self, exception)
		self._loop.call_soon(self._tb_logger.activate)
```
### 从其他的 future 复制状态
把目标 future 的状态以及结果复制到当前 future 上。
目标 future 可能是 concurrent.futures.Future 对象
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
### 支持迭代
```python
def __iter__(self):
	if not self.done():
		self._blocking = True
		yield self  # This tells Task to wait for completion.
	assert self.done(), "yield from wasn't used with future"
	return self.result()
```
## 关联 future
把 concurrent.futures.Future 对象关联到 asyncio 的 future 对象上
如果新的future被取消旧的future也会被取消，旧的future完成后会把结果和状态复制到新的future上。
```python
def wrap_future(fut, *, loop=None):
    if isinstance(fut, Future):
        return fut
    assert isinstance(fut, concurrent.futures.Future), \
        'concurrent.futures.Future is expected, got {!r}'.format(fut)
	if loop is None:
        loop = events.get_event_loop()
    new_future = Future(loop=loop)
    def _check_cancel_other(f):
        if f.cancelled():
            fut.cancel()
    new_future.add_done_callback(_check_cancel_other)
    fut.add_done_callback(
        lambda future: loop.call_soon_threadsafe(
            new_future._copy_state, future))
    return new_future
```
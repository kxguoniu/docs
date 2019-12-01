[TOC]
# asyncio 之 futures.py
# head
文件主要实现了一个 Future 类和一个 wrap_future 方法
Future 类中主要的方法有 设置结果、设置异常、获取结果、获取异常、添加回调函数、删除回调函数、取消future等。
wrap_future 方法主要作用是把其他模块的 future 对象关联到当前模块的 future 对象上，实现 future 对象之间的状态和内容同步。
## 文件内常量
文件开头定义了三种 future 状态常量。分别是待执行、已取消、已完成。
三种异常分别是取消异常、操作异常、超时异常。还有一个判断 future 的方法。
```python
_PENDING = base_futures._PENDING
_CANCELLED = base_futures._CANCELLED
_FINISHED = base_futures._FINISHED

CancelledError = base_futures.CancelledError
InvalidStateError = base_futures.InvalidStateError
TimeoutError = base_futures.TimeoutError
isfuture = base_futures.isfuture

STACK_DEBUG = logging.DEBUG - 1  # heavy-duty debugging
```
## class Future
### 类变量
实例初始化的时候的默认值
```python
class Future:
	_state = _PENDING
    _result = None
    _exception = None
    _loop = None
    _source_traceback = None
	# 使用 yield from future 或者 await future 的时候这个状态会设置为True表示 future 被阻塞。
    _asyncio_future_blocking = False
	# 这个属性在给future设置异常的时候被设置为True
	# 如果在future销毁之前协程没有调用future对象的 result 或者 exception 方法
	# 也就是没有使用future的结果，则会记录异常
    _log_traceback = False
```
### 实例初始化
`future`实例初始化除了默认的类变量还有一个可调用对象列表的属性。这个列表中的对象会在 future 取消或者完成之后依次调用。
```python
def __init__(self, *, loop=None):
	if loop is None:
		self._loop = events.get_event_loop()
	else:
		self._loop = loop
	self._callbacks = []
	if self._loop.get_debug():
		self._source_traceback = format_helpers.extract_stack(sys._getframe(1))
```
### 删除 future
当`future`的`__log_traceback`为真时记录异常信息
```python
def __del__(self):
	if not self.__log_traceback:
		return
	exc = self._exception
	context = {
		'message':
			f'{self.__class__.__name__} exception was never retrieved',
		'exception': exc,
		'future': self,
	}
	if self._source_traceback:
		context['source_traceback'] = self._source_traceback
	self._loop.call_exception_handler(context)
```
### set __log_traceback
只能把`__log_traceback`属性设置为False，让 future 在被销毁时不记录日志
```python
@property
def _log_traceback(self):
	return self.__log_traceback

@_log_traceback.setter
def _log_traceback(self, val):
	if bool(val):
		raise ValueError('_log_traceback can only be set to False')
	self.__log_traceback = False
```
### 取消 future
把 future 对象的状态设置为取消并依次执行回调函数列表中的对象。
```python
def cancel(self):
	# future 被取消，销毁时不记录异常
	self.__log_traceback = False
	if self._state != _PENDING:
		return False
	self._state = _CANCELLED
	self.__schedule_callbacks()
	return True
# 把需要调用的对象依次放进 loop 循环中执行，并清空列表
def __schedule_callbacks(self):
	callbacks = self._callbacks[:]
	if not callbacks:
		return

	self._callbacks[:] = []
	for callback, ctx in callbacks:
		self._loop.call_soon(callback, self, context=ctx)
```
### future 的状态
获取`future`的`loop`和判断 future 是否取消或者完成
```python
def get_loop(self):
	return self._loop

def cancelled(self):
	return self._state == _CANCELLED

def done(self):
	return self._state != _PENDING
```
### 获取 future 的结果
如果`future`关联的任务执行结束则返回其结果，如果有异常信息则抛出
```python
def result(self):
	if self._state == _CANCELLED:
		raise CancelledError
	if self._state != _FINISHED:
		raise InvalidStateError('Result is not ready.')
	# future 的结果已被使用，删除时不记录异常
	self.__log_traceback = False
	# 如果有异常信息则抛出异常信息
	if self._exception is not None:
		raise self._exception
	return self._result
```
### 获取 future 的异常信息
`future`完成后获取产生的异常信息，这个与获取结果不同，获取结果是抛出异常信息。
```python
def exception(self):
	if self._state == _CANCELLED:
		raise CancelledError
	if self._state != _FINISHED:
		raise InvalidStateError('Exception is not set.')
	# future 异常被使用，删除时不记录异常
	self.__log_traceback = False
	return self._exception
```
### 添加回调对象
给`future`添加回调函数，如果`future`已经完成则立刻执行回调函数。
```python
def add_done_callback(self, fn, *, context=None):
	if self._state != _PENDING:
		self._loop.call_soon(fn, self, context=context)
	else:
		if context is None:
			context = contextvars.copy_context()
		self._callbacks.append((fn, context))
```
### 删除回调对象
删除`future`回调列表中的回调函数并返回删除的数量
```python
def remove_done_callback(self, fn):
	filtered_callbacks = [f for f in self._callbacks if f != fn]
	removed_count = len(self._callbacks) - len(filtered_callbacks)
	if removed_count:
		self._callbacks[:] = filtered_callbacks
	return removed_count
```
### 设置 future 结果
`future`关联的任务已经完成，给`future`设置结果，更新状态，依次执行回调对象
```python
def set_result(self, result):
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	self._result = result
	self._state = _FINISHED
	self.__schedule_callbacks()
```
### 设置 future 异常
同设置结果
```python
def set_exception(self, exception):
	if self._state != _PENDING:
		raise InvalidStateError('{}: {!r}'.format(self._state, self))
	if isinstance(exception, type):
		exception = exception()
	if type(exception) is StopIteration:
		raise TypeError("StopIteration interacts badly with generators "
						"and cannot be raised into a Future")
	self._exception = exception
	self._state = _FINISHED
	self.__schedule_callbacks()
	# 协程执行出现了异常，如果这个异常没有被消费，则在删除future时记录信息
	self.__log_traceback = True
```
### yield from or await
```python
def __await__(self):
	if not self.done():
		self._asyncio_future_blocking = True
		yield self  # This tells Task to wait for completion.
	if not self.done():
		raise RuntimeError("await wasn't used with future")
	return self.result()

__iter__ = __await__
```
## 文件内置方法
### 获取执行 future 的 loop
兼容之前的版本
```python
def _get_loop(fut):
    try:
        get_loop = fut.get_loop
    except AttributeError:
        pass
    else:
        return get_loop()
    return fut._loop
```
### 给 future 设置结果
如果`future`没有取消就设置结果
```python
def _set_result_unless_cancelled(fut, result):
    if fut.cancelled():
        return
    fut.set_result(result)
```
### 拷贝状态1
拷贝一个已完成`future`的状态到`concurrent.futures.Future`对象中。
```python
def _set_concurrent_future_state(concurrent, source):
    assert source.done()
    if source.cancelled():
        concurrent.cancel()
    if not concurrent.set_running_or_notify_cancel():
        return
    exception = source.exception()
    if exception is not None:
        concurrent.set_exception(exception)
    else:
        result = source.result()
        concurrent.set_result(result)
```
### 拷贝状态2
拷贝一个已完成`future`的状态到另一个`future`中
```python
def _copy_future_state(source, dest):
    assert source.done()
    if dest.cancelled():
        return
    assert not dest.done()
    if source.cancelled():
        dest.cancel()
    else:
        exception = source.exception()
        if exception is not None:
            dest.set_exception(exception)
        else:
            result = source.result()
            dest.set_result(result)
```
### 链接两个 future
链接两个`future`当一个完成时另一个也会完成
```python
def _chain_future(source, destination):
    if not isfuture(source) and not isinstance(source,
                                               concurrent.futures.Future):
        raise TypeError('A future is required for source argument')
    if not isfuture(destination) and not isinstance(destination,
                                                    concurrent.futures.Future):
        raise TypeError('A future is required for destination argument')
    source_loop = _get_loop(source) if isfuture(source) else None
    dest_loop = _get_loop(destination) if isfuture(destination) else None

    def _set_state(future, other):
        if isfuture(future):
            _copy_future_state(other, future)
        else:
            _set_concurrent_future_state(future, other)

    def _call_check_cancel(destination):
        if destination.cancelled():
            if source_loop is None or source_loop is dest_loop:
                source.cancel()
            else:
                source_loop.call_soon_threadsafe(source.cancel)

    def _call_set_state(source):
        if (destination.cancelled() and
                dest_loop is not None and dest_loop.is_closed()):
            return
        if dest_loop is None or dest_loop is source_loop:
            _set_state(destination, source)
        else:
            dest_loop.call_soon_threadsafe(_set_state, destination, source)
	# 目标future设置回调函数，目标future取消之后源future也要取消
    destination.add_done_callback(_call_check_cancel)
	# 源future设置回调函数，源完成或者取消都要拷贝到目标上
    source.add_done_callback(_call_set_state)
```
## 关联 future
把 concurrent.futures.Future 对象关联到 asyncio 的 future 对象上
如果新的future被取消旧的future也会被取消，旧的future完成后会把结果和状态复制到新的future上。
```python
def wrap_future(future, *, loop=None):
    if isfuture(future):
        return future
    assert isinstance(future, concurrent.futures.Future), \
        f'concurrent.futures.Future is expected, got {future!r}'
    if loop is None:
        loop = events.get_event_loop()
    new_future = loop.create_future()
    _chain_future(future, new_future)
    return new_future
```
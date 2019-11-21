# asyncio 之 events.py
## Handler
注册回调方法返回的对象
### 初始化
```python
class Handle:
	# 允许修改的属性
    __slots__ = ('_callback', '_args', '_cancelled', '_loop',
                 '_source_traceback', '_repr', '__weakref__')

    def __init__(self, callback, args, loop):
		# 回调函数不能是一个 Handle 对象
        assert not isinstance(callback, Handle), 'A Handle is not a callback'
        self._loop = loop
        self._callback = callback
        self._args = args
		# 如果已经取消则为真
        self._cancelled = False
        self._repr = None
        if self._loop.get_debug():
            self._source_traceback = traceback.extract_stack(sys._getframe(1))
        else:
            self._source_traceback = None
```
### 取消和运行
```python
def cancel(self):
	# 没有被取消
	if not self._cancelled:
		# 设置取消状态为 True
		self._cancelled = True
		if self._loop.get_debug():
			# Keep a representation in debug mode to keep callback and
			# parameters. For example, to log the warning
			# "Executing <Handle...> took 2.5 second"
			self._repr = repr(self)
		# 重置回调函数和其参数
		self._callback = None
		self._args = None

def _run(self):
	# 尝试调用回调函数
	try:
		self._callback(*self._args)
	except Exception as exc:
		cb = _format_callback(self._callback, self._args)
		msg = 'Exception in callback {}'.format(cb)
		context = {
			'message': msg,
			'exception': exc,
			'handle': self,
		}
		if self._source_traceback:
			context['source_traceback'] = self._source_traceback
		# 调用异常处理器处理异常
		self._loop.call_exception_handler(context)
	self = None
```
## TimerHandle
注册定时回调方法返回的对象
### 代码
```python
class TimerHandle(Handle):
    __slots__ = ['_scheduled', '_when']

    def __init__(self, when, callback, args, loop):
        assert when is not None
        super().__init__(callback, args, loop)
        if self._source_traceback:
            del self._source_traceback[-1]
		# 在什么时间调用
        self._when = when
		# TODO
        self._scheduled = False

    def __hash__(self):
        return hash(self._when)

    def __lt__(self, other):
        return self._when < other._when

    def __le__(self, other):
        if self._when < other._when:
            return True
        return self.__eq__(other)

    def __gt__(self, other):
        return self._when > other._when

    def __ge__(self, other):
        if self._when > other._when:
            return True
        return self.__eq__(other)

    def __eq__(self, other):
        if isinstance(other, TimerHandle):
            return (self._when == other._when and
                    self._callback == other._callback and
                    self._args == other._args and
                    self._cancelled == other._cancelled)
        return NotImplemented

    def __ne__(self, other):
        equal = self.__eq__(other)
        return NotImplemented if equal is NotImplemented else not equal

    def cancel(self):
        if not self._cancelled:
            self._loop._timer_handle_cancelled(self)
        super().cancel()
```
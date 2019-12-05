[TOC]
## 摘要
## class Handler
注册回调方法后返回一个`handle`对象。
### 初始化
```python
class Handle:
    __slots__ = ('_callback', '_args', '_cancelled', '_loop',
                 '_source_traceback', '_repr', '__weakref__',
                 '_context')
    def __init__(self, callback, args, loop, context=None):
        if context is None:
            context = contextvars.copy_context()
        self._context = context
        self._loop = loop
        self._callback = callback
        self._args = args
		# 如果已经取消则设置为真
        self._cancelled = False
        self._repr = None
        if self._loop.get_debug():
            self._source_traceback = format_helpers.extract_stack(
                sys._getframe(1))
        else:
            self._source_traceback = None
```
### 取消
```python
def cancelled(self):
	return self._cancelled

def cancel(self):
	if not self._cancelled:
		self._cancelled = True
		if self._loop.get_debug():
			self._repr = repr(self)
		# 回调函数和其参数都设置为None
		self._callback = None
		self._args = None
```
### 运行
```python
def _run(self):
	try:
		self._context.run(self._callback, *self._args)
	except Exception as exc:
		cb = format_helpers._format_callback_source(
			self._callback, self._args)
		msg = f'Exception in callback {cb}'
		context = {
			'message': msg,
			'exception': exc,
			'handle': self,
		}
		if self._source_traceback:
			context['source_traceback'] = self._source_traceback
		# 调用异常处理器处理异常
		self._loop.call_exception_handler(context)
	self = None  # Needed to break cycles when an exception occurs.
```
## class TimerHandle
注册定时回调方法后返回一个`timerhandle`对象。
```python
class TimerHandle(Handle):
    __slots__ = ['_scheduled', '_when']
    def __init__(self, when, callback, args, loop, context=None):
        assert when is not None
        super().__init__(callback, args, loop, context)
        if self._source_traceback:
            del self._source_traceback[-1]
		# 什么时候执行
        self._when = when
		# 在创建的定时函数的时候这个属性被设置为True
		# 当timerhandle对象被取消或者被添加到loop的待执行队列中的时候会被设置为False
		# 表示任务已取消，或者正在准备执行/已执行。
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

	# 取消定时任务，把loop中的任务取消数量增加一
    def cancel(self):
        if not self._cancelled:
            self._loop._timer_handle_cancelled(self)
        super().cancel()

	# 返回在什么时候执行回电函数
    def when(self):
        return self._when
```
## 抽象服务器
### AbstractServer
## 抽象事件循环
### AbstractEventLoop
## 抽象事件循环策略
### AbstractEventLoopPolicy
## 基础事件循环策略
在这个策略中，每个线程都有自己的事件循环。但是，默认情况下我们只会为主线程自动创建一个事件循环;其他线程在默认情况下没有事件循环。
### 初始化
```python
class BaseDefaultEventLoopPolicy(AbstractEventLoopPolicy):
	# 其应该是一个可调用对象,返回值是一个 event loop
    _loop_factory = None
	# 线程全局变量,每个线程都可以使用,但线程之间的数据互不影响
    class _Local(threading.local):
        _loop = None		# 当前线程的 loop
        _set_called = False # 标识 loop 是否已设置

    def __init__(self):
        self._local = self._Local()
```
### 获取/设置/新建 event loop
```python
def get_event_loop(self):
	# 如果是主线程并且没有设置 loop
	if (self._local._loop is None and
		not self._local._set_called and
		isinstance(threading.current_thread(), threading._MainThread)):
		self.set_event_loop(self.new_event_loop())
	if self._local._loop is None:
		raise RuntimeError('There is no current event loop in thread %r.'
						   % threading.current_thread().name)
	return self._local._loop

# 设置当前线程的 event_loop
def set_event_loop(self, loop):
	self._local._set_called = True
	assert loop is None or isinstance(loop, AbstractEventLoop)
	self._local._loop = loop
# 从 loop_factory 里面获取一个 event loop 对象
def new_event_loop(self):
	return self._loop_factory()
```
## 文件内函数
```python
_event_loop_policy = None
# 如果没有设置事件循环策略,给一个默认的事件循环策略
_lock = threading.Lock()
def _init_event_loop_policy():
    global _event_loop_policy
    with _lock:
        if _event_loop_policy is None:
            from . import DefaultEventLoopPolicy
            _event_loop_policy = DefaultEventLoopPolicy()

# 获取当前的事件循环策略
def get_event_loop_policy():
    if _event_loop_policy is None:
        _init_event_loop_policy()
    return _event_loop_policy

# 设置当前的事件循环策略
def set_event_loop_policy(policy):
    global _event_loop_policy
    assert policy is None or isinstance(policy, AbstractEventLoopPolicy)
    _event_loop_policy = policy

# 获取当前 event_loop
def get_event_loop():
    return get_event_loop_policy().get_event_loop()

# 设置当前 event_loop
def set_event_loop(loop):
    get_event_loop_policy().set_event_loop(loop)

# 获取新的 event_loop
def new_event_loop():
    return get_event_loop_policy().new_event_loop()

def get_child_watcher():
    return get_event_loop_policy().get_child_watcher()

def set_child_watcher(watcher):
    return get_event_loop_policy().set_child_watcher(watcher)
```
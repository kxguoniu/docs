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
		# 在创建时间回调对象之后会把这个属性设置为真
		# 在对象取消或者执行后会设置为假
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

	# 取消定时任务,把 loop 中取消的时间回调数量增加1
    def cancel(self):
        if not self._cancelled:
            self._loop._timer_handle_cancelled(self)
        super().cancel()
```
## 抽象服务器
### AbstractServer
## 抽象事件循环
### AbstractEventLoop
## 抽象事件循环策略
### AbstractEventLoopPolicy
## 基础事件循环策略
在这个策略中，每个线程都有自己的事件循环。但是，默认情况下我们只会为主线程自动创建一个事件循环;其他线程在默认情况下没有事件循环。
```python
class BaseDefaultEventLoopPolicy(AbstractEventLoopPolicy):
	# 其应该是一个可调用对象,返回值是一个 event loop
    _loop_factory = None
	# 线程全局变量,每个线程都可以使用,但线程之间的数据互不影响
    class _Local(threading.local):
        _loop = None
        _set_called = False

    def __init__(self):
        self._local = self._Local()

    def get_event_loop(self):
		# 如果没有 loop 并且设置状态 False,且该线程是主线程,设置一个loop
        if (self._local._loop is None and
            not self._local._set_called and
            isinstance(threading.current_thread(), threading._MainThread)):
            self.set_event_loop(self.new_event_loop())
        if self._local._loop is None:
            raise RuntimeError('There is no current event loop in thread %r.'
                               % threading.current_thread().name)
		# 返回当前线程的 event loop
        return self._local._loop
	# 设置当前线程的 event_loop
    def set_event_loop(self, loop):
        self._local._set_called = True
        assert loop is None or isinstance(loop, AbstractEventLoop)
        self._local._loop = loop
	# 从 loop_factory 里面获取一个 event loop
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
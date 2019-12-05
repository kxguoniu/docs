[TOC]
## 摘要
`coroutines.py`中定义了那些类型可以作为协程，并实现了一个协程装饰器`coroutine`。此外还有判断一个对象是不是协程的方法`iscoroutine`以及判断一个函数是不是返回协程的方法`iscoroutinefunction`。
## 文件内置属性和方法
```python
def _is_debug_mode():
    return sys.flags.dev_mode or (not sys.flags.ignore_environment and
                                  bool(os.environ.get('PYTHONASYNCIODEBUG')))

_DEBUG = _is_debug_mode()
_is_coroutine = object()
_COROUTINE_TYPES = (types.CoroutineType, types.GeneratorType,
                    collections.abc.Coroutine, CoroWrapper)
_iscoroutine_typecache = set()
```
## class CoroWrapper
在DEBUG模式下包装协程。
```python
class CoroWrapper:
    def __init__(self, gen, func=None):
        assert inspect.isgenerator(gen) or inspect.iscoroutine(gen), gen
        self.gen = gen
        self.func = func  # Used to unwrap @coroutine decorator
        self._source_traceback = format_helpers.extract_stack(sys._getframe(1))
        self.__name__ = getattr(gen, '__name__', None)
        self.__qualname__ = getattr(gen, '__qualname__', None)

    def __repr__(self):
        coro_repr = _format_coroutine(self)
        if self._source_traceback:
            frame = self._source_traceback[-1]
            coro_repr += f', created at {frame[0]}:{frame[1]}'

        return f'<{self.__class__.__name__} {coro_repr}>'

    def __iter__(self):
        return self

    def __next__(self):
        return self.gen.send(None)

    def send(self, value):
        return self.gen.send(value)

    def throw(self, type, value=None, traceback=None):
        return self.gen.throw(type, value, traceback)

    def close(self):
        return self.gen.close()

    @property
    def gi_frame(self):
        return self.gen.gi_frame

    @property
    def gi_running(self):
        return self.gen.gi_running

    @property
    def gi_code(self):
        return self.gen.gi_code

    def __await__(self):
        return self

    @property
    def gi_yieldfrom(self):
        return self.gen.gi_yieldfrom

    def __del__(self):
        # Be careful accessing self.gen.frame -- self.gen might not exist.
        gen = getattr(self, 'gen', None)
        frame = getattr(gen, 'gi_frame', None)
        if frame is not None and frame.f_lasti == -1:
            msg = f'{self!r} was never yielded from'
            tb = getattr(self, '_source_traceback', ())
            if tb:
                tb = ''.join(traceback.format_list(tb))
                msg += (f'\nCoroutine object created at '
                        f'(most recent call last, truncated to '
                        f'{constants.DEBUG_STACK_DEPTH} last lines):\n')
                msg += tb.rstrip()
            logger.error(msg)

```
## 文件内方法
### def coroutine
```python
def coroutine(func):
	# 如果参数是一个返回协程的函数，直接返回。async def
    if inspect.iscoroutinefunction(func):
        return func

	# 如果参数是一个生成器函数
    if inspect.isgeneratorfunction(func):
        coro = func
    else:
        @functools.wraps(func)
        def coro(*args, **kw):
            res = func(*args, **kw)
            if (base_futures.isfuture(res) or inspect.isgenerator(res) or
                    isinstance(res, CoroWrapper)):
                res = yield from res
            else:
                # If 'res' is an awaitable, run it.
                try:
                    await_meth = res.__await__
                except AttributeError:
                    pass
                else:
                    if isinstance(res, collections.abc.Awaitable):
                        res = yield from await_meth()
            return res

	# 把传递的参数包装成协程。
    coro = types.coroutine(coro)
    if not _DEBUG:
        wrapper = coro
    else:
        @functools.wraps(func)
        def wrapper(*args, **kwds):
            w = CoroWrapper(coro(*args, **kwds), func=func)
            if w._source_traceback:
                del w._source_traceback[-1]
            w.__name__ = getattr(func, '__name__', None)
            w.__qualname__ = getattr(func, '__qualname__', None)
            return w

	# 给返回的装饰器打上协程的标签
    wrapper._is_coroutine = _is_coroutine  # For iscoroutinefunction().
    return wrapper
```
### def iscoroutinefunction
```python
def iscoroutinefunction(func):
    return (inspect.iscoroutinefunction(func) or
            getattr(func, '_is_coroutine', None) is _is_coroutine)
```
### def iscoroutine
```python
def iscoroutine(obj):
	# 如果对象类型在缓存之中，直接返回
    if type(obj) in _iscoroutine_typecache:
        return True

	# 如果对象是协程对象，缓存其类型。缓存的协程类型不能超过100个
    if isinstance(obj, _COROUTINE_TYPES):
        if len(_iscoroutine_typecache) < 100:
            _iscoroutine_typecache.add(type(obj))
        return True
    else:
        return False
```
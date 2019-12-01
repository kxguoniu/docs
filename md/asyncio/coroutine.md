# asyncio 之 coroutines.py
这个文件里面都是垃圾
## TODO
### debug
```python
def _is_debug_mode():
    return sys.flags.dev_mode or (not sys.flags.ignore_environment and
                                  bool(os.environ.get('PYTHONASYNCIODEBUG')))

_DEBUG = _is_debug_mode()
```
### coroutine
```python
def coroutine(func):
    if inspect.iscoroutinefunction(func):
        return func

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

    coro = types.coroutine(coro)
    if not _DEBUG:
        wrapper = coro
    else:
        @functools.wraps(func)
        def wrapper(*args, **kwds):
            w = CoroWrapper(coro(*args, **kwds), func=func)
            if w._source_traceback:
                del w._source_traceback[-1]
            # Python < 3.5 does not implement __qualname__
            # on generator objects, so we set it manually.
            # We use getattr as some callables (such as
            # functools.partial may lack __qualname__).
            w.__name__ = getattr(func, '__name__', None)
            w.__qualname__ = getattr(func, '__qualname__', None)
            return w

    wrapper._is_coroutine = _is_coroutine  # For iscoroutinefunction().
    return wrapper
```
### iscoroutine
判断一个对象是不是协程对象
```python
_COROUTINE_TYPES = (types.CoroutineType, types.GeneratorType,
                    collections.abc.Coroutine, CoroWrapper)
_iscoroutine_typecache = set()

def iscoroutine(obj):
    if type(obj) in _iscoroutine_typecache:
        return True

    if isinstance(obj, _COROUTINE_TYPES):
        if len(_iscoroutine_typecache) < 100:
            _iscoroutine_typecache.add(type(obj))
        return True
    else:
        return False
```
### iscoroutinefuction
判断一个对象是不是协程函数
```python
_is_coroutine = object()
def iscoroutinefunction(func):
    """Return True if func is a decorated coroutine function."""
    return (inspect.iscoroutinefunction(func) or
            getattr(func, '_is_coroutine', None) is _is_coroutine)
```

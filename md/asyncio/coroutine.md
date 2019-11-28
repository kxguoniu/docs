# asyncio 之 coroutines.py
这个文件里面都是垃圾
## TODO
### coroutine
```python
def coroutine(func):
    """Decorator to mark coroutines.

    If the coroutine is not yielded from before it is destroyed,
    an error message is logged.
    """
    if inspect.isgeneratorfunction(func):
        coro = func
    else:
        @functools.wraps(func)
        def coro(*args, **kw):
            res = func(*args, **kw)
            if isinstance(res, futures.Future) or inspect.isgenerator(res):
                res = yield from res
            return res

    if not _DEBUG:
        wrapper = coro
    else:
        @functools.wraps(func)
        def wrapper(*args, **kwds):
            w = CoroWrapper(coro(*args, **kwds), func)
            if w._source_traceback:
                del w._source_traceback[-1]
            w.__name__ = func.__name__
            if hasattr(func, '__qualname__'):
                w.__qualname__ = func.__qualname__
            w.__doc__ = func.__doc__
            return w

    wrapper._is_coroutine = True  # For iscoroutinefunction().
    return wrapper
```
### iscoroutine
判断一个对象是不是协程对象
```python
_COROUTINE_TYPES = (types.GeneratorType, CoroWrapper)
def iscoroutine(obj):
    return isinstance(obj, _COROUTINE_TYPES)
```
### iscoroutinefuction
判断一个对象是不是
```python
def iscoroutinefunction(func):
    """Return True if func is a decorated coroutine function."""
    return getattr(func, '_is_coroutine', False)
```

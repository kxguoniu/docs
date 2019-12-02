[TOC]
## 摘要
这个文件中定义了`Future`类中的四种异常，三种状态以及一个判断对象是不是`future`对象的方法。
## code
```python
Error = concurrent.futures._base.Error
CancelledError = concurrent.futures.CancelledError
TimeoutError = concurrent.futures.TimeoutError

class InvalidStateError(Error):
    """The operation is not allowed in this state."""

# States for Future.
_PENDING = 'PENDING'
_CANCELLED = 'CANCELLED'
_FINISHED = 'FINISHED'

def isfuture(obj):
    return (hasattr(obj.__class__, '_asyncio_future_blocking') and
            obj._asyncio_future_blocking is not None)
```
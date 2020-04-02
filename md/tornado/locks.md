# tornado 之 locks.md
## 摘要

import collections
from concurrent.futures import CancelledError
import datetime
import types

from tornado import gen, ioloop
from tornado.concurrent import Future, future_set_result_unless_cancelled

from typing import Union, Optional, Type, Any, Awaitable
import typing

if typing.TYPE_CHECKING:
    from typing import Deque, Set  # noqa: F401

__all__ = ["Condition", "Event", "Semaphore", "BoundedSemaphore", "Lock"]

## class _TimeoutGarbageCollector
```python
class _TimeoutGarbageCollector(object):
	# 定时清理超时的等待器
    """Base class for objects that periodically clean up timed-out waiters.

    Avoids memory leak in a common pattern like:

        while True:
            yield condition.wait(short_timeout)
            print('looping....')
    """

    def __init__(self) -> None:
    	# 等待器的队列
        self._waiters = collections.deque()  # type: Deque[Future]
        # 取消等待的数量
        self._timeouts = 0

    def _garbage_collect(self) -> None:
        # 偶尔清理超时的协程
        self._timeouts += 1
        # 取消数量超过阈值，清理
        if self._timeouts > 100:
            self._timeouts = 0
            self._waiters = collections.deque(w for w in self._waiters if not w.done())
```
## class Condition
```python
class Condition(_TimeoutGarbageCollector):
	# 条件变量
	# 和线程的条件变量相似但不需要获取和释放低层锁

    def __init__(self) -> None:
        super(Condition, self).__init__()
        self.io_loop = ioloop.IOLoop.current()


    def wait(self, timeout: Union[float, datetime.timedelta] = None) -> Awaitable[bool]:
    	# 阻塞协程直到被通知
        waiter = Future()  # type: Future[bool]
        self._waiters.append(waiter)
        if timeout:

            def on_timeout() -> None:
                if not waiter.done():
                    future_set_result_unless_cancelled(waiter, False)
                self._garbage_collect()

            io_loop = ioloop.IOLoop.current()
            timeout_handle = io_loop.add_timeout(timeout, on_timeout)
            waiter.add_done_callback(lambda _: io_loop.remove_timeout(timeout_handle))
        return waiter

    def notify(self, n: int = 1) -> None:
        # 通知 n 个被阻塞的协程
        waiters = []  # Waiters we plan to run right now.
        while n and self._waiters:
            waiter = self._waiters.popleft()
            if not waiter.done():  # Might have timed out.
                n -= 1
                waiters.append(waiter)

        for waiter in waiters:
            future_set_result_unless_cancelled(waiter, True)

    def notify_all(self) -> None:
        # 唤醒所有被阻塞的协程
        self.notify(len(self._waiters))
```

## class Event
### 初始化
```python
class Event(object):
    # 事件变量，和线程的事件类似
    def __init__(self) -> None:
        self._value = False
        self._waiters = set()  # type: Set[Future[None]]

    def is_set(self) -> bool:
        # 返回事件属性
        return self._value

    def set(self) -> None:
        # 唤醒所有阻塞的协程
        if not self._value:
            self._value = True

            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(None)

    def clear(self) -> None:
        # 重置事件
        self._value = False

    def wait(self, timeout: Union[float, datetime.timedelta] = None) -> Awaitable[None]:
        # 阻塞协程直到事件被设置，或者抛出超时异常
        fut = Future()  # type: Future[None]
        if self._value:
            fut.set_result(None)
            return fut
        self._waiters.add(fut)
        fut.add_done_callback(lambda fut: self._waiters.remove(fut))
        if timeout is None:
            return fut
        else:
            timeout_fut = gen.with_timeout(
                timeout, fut, quiet_exceptions=(CancelledError,)
            )
            # This is a slightly clumsy workaround for the fact that
            # gen.with_timeout doesn't cancel its futures. Cancelling
            # fut will remove it from the waiters list.
            timeout_fut.add_done_callback(
                lambda tf: fut.cancel() if not fut.done() else None
            )
            return timeout_fut
```

## class _ReleasingContextManager
```python
class _ReleasingContextManager(object):
    # with 上下文管理
    def __init__(self, obj: Any) -> None:
        self._obj = obj

    def __enter__(self) -> None:
        pass

    def __exit__(
        self,
        exc_type: "Optional[Type[BaseException]]",
        exc_val: Optional[BaseException],
        exc_tb: Optional[types.TracebackType],
    ) -> None:
        self._obj.release()
```

## class Semaphore
信号量
```python
class Semaphore(_TimeoutGarbageCollector):
    def __init__(self, value: int = 1) -> None:
        super(Semaphore, self).__init__()
        if value < 0:
            raise ValueError("semaphore initial value must be >= 0")
        # 信号数量
        self._value = value

    def release(self) -> None:
        # 增加信号并尝试唤醒一个阻塞的协程
        self._value += 1
        while self._waiters:
            waiter = self._waiters.popleft()
            if not waiter.done():
                self._value -= 1
                waiter.set_result(_ReleasingContextManager(self))
                break

    def acquire(
        self, timeout: Union[float, datetime.timedelta] = None
    ) -> Awaitable[_ReleasingContextManager]:
        # 如果计数器为0则被阻塞直到获取到信号
        waiter = Future()  # type: Future[_ReleasingContextManager]
        if self._value > 0:
            self._value -= 1
            waiter.set_result(_ReleasingContextManager(self))
        else:
            self._waiters.append(waiter)
            if timeout:

                def on_timeout() -> None:
                    if not waiter.done():
                        waiter.set_exception(gen.TimeoutError())
                    self._garbage_collect()

                io_loop = ioloop.IOLoop.current()
                timeout_handle = io_loop.add_timeout(timeout, on_timeout)
                waiter.add_done_callback(
                    lambda _: io_loop.remove_timeout(timeout_handle)
                )
        return waiter

    def __enter__(self) -> None:
        raise RuntimeError("Use 'async with' instead of 'with' for Semaphore")

    def __exit__(
        self,
        typ: "Optional[Type[BaseException]]",
        value: Optional[BaseException],
        traceback: Optional[types.TracebackType],
    ) -> None:
        self.__enter__()

    async def __aenter__(self) -> None:
        await self.acquire()

    async def __aexit__(
        self,
        typ: "Optional[Type[BaseException]]",
        value: Optional[BaseException],
        tb: Optional[types.TracebackType],
    ) -> None:
        self.release()
```
## class BoundedSemaphore
有界信号量
```python
class BoundedSemaphore(Semaphore):
    def __init__(self, value: int = 1) -> None:
        super(BoundedSemaphore, self).__init__(value=value)
        self._initial_value = value

    def release(self) -> None:
        if self._value >= self._initial_value:
            raise ValueError("Semaphore released too many times")
        super(BoundedSemaphore, self).release()
```
## class Lock
互斥锁
```python
class Lock(object):
    def __init__(self) -> None:
        self._block = BoundedSemaphore(value=1)

    def acquire(
        self, timeout: Union[float, datetime.timedelta] = None
    ) -> Awaitable[_ReleasingContextManager]:
        # 获取锁
        return self._block.acquire(timeout)

    def release(self) -> None:
       # 释放锁
        try:
            self._block.release()
        except ValueError:
            raise RuntimeError("release unlocked lock")

    def __enter__(self) -> None:
        raise RuntimeError("Use `async with` instead of `with` for Lock")

    def __exit__(
        self,
        typ: "Optional[Type[BaseException]]",
        value: Optional[BaseException],
        tb: Optional[types.TracebackType],
    ) -> None:
        self.__enter__()

    async def __aenter__(self) -> None:
        await self.acquire()

    async def __aexit__(
        self,
        typ: "Optional[Type[BaseException]]",
        value: Optional[BaseException],
        tb: Optional[types.TracebackType],
    ) -> None:
        self.release()
```
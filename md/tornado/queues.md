[TOC]
# tornado 之 queues.py
## 摘要
非线程安全

import collections
import datetime
import heapq

from tornado import gen, ioloop
from tornado.concurrent import Future, future_set_result_unless_cancelled
from tornado.locks import Event

from typing import Union, TypeVar, Generic, Awaitable
import typing

if typing.TYPE_CHECKING:
    from typing import Deque, Tuple, List, Any  # noqa: F401

_T = TypeVar("_T")

__all__ = ["Queue", "PriorityQueue", "LifoQueue", "QueueFull", "QueueEmpty"]


class QueueEmpty(Exception):
    """Raised by `.Queue.get_nowait` when the queue has no items."""

    pass


class QueueFull(Exception):
    """Raised by `.Queue.put_nowait` when a queue is at its maximum size."""

    pass


def _set_timeout(
    future: Future, timeout: Union[None, float, datetime.timedelta]
) -> None:
    if timeout:

        def on_timeout() -> None:
            if not future.done():
                future.set_exception(gen.TimeoutError())

        io_loop = ioloop.IOLoop.current()
        timeout_handle = io_loop.add_timeout(timeout, on_timeout)
        future.add_done_callback(lambda _: io_loop.remove_timeout(timeout_handle))


class _QueueIterator(Generic[_T]):
    def __init__(self, q: "Queue[_T]") -> None:
        self.q = q

    def __anext__(self) -> Awaitable[_T]:
        return self.q.get()

## class Queue
```python
class Queue(Generic[_T]):
    # Exact type depends on subclass. Could be another generic
    # parameter and use protocols to be more precise here.
    _queue = None  # type: Any

    def __init__(self, maxsize: int = 0) -> None:
        if maxsize is None:
            raise TypeError("maxsize can't be None")

        if maxsize < 0:
            raise ValueError("maxsize can't be negative")
        # 队列最大值
        self._maxsize = maxsize
        # 实例化队列
        self._init()
        # 被阻塞的等待消费数据的协程
        self._getters = collections.deque([])  # type: Deque[Future[_T]]
        # 被阻塞的等待伸长数据的协程
        self._putters = collections.deque([])  # type: Deque[Tuple[_T, Future[None]]]
        # 未完成的任务
        self._unfinished_tasks = 0
        # 事件锁
        self._finished = Event()
        self._finished.set()

    @property
    def maxsize(self) -> int:
        return self._maxsize

    def qsize(self) -> int:
        return len(self._queue)

    def empty(self) -> bool:
        return not self._queue

    def full(self) -> bool:
        if self.maxsize == 0:
            return False
        else:
            return self.qsize() >= self.maxsize

    def put(
        self, item: _T, timeout: Union[float, datetime.timedelta] = None
    ) -> "Future[None]":
        # 把数据放入到队列，如果队列已满则阻塞
        future = Future()  # type: Future[None]
        try:
            self.put_nowait(item)
        except QueueFull:
            self._putters.append((item, future))
            _set_timeout(future, timeout)
        else:
            future.set_result(None)
        return future

    def put_nowait(self, item: _T) -> None:
        # 非阻塞的把数据放入到队列
        self._consume_expired()
        if self._getters:
            assert self.empty(), "queue non-empty, why are getters waiting?"
            getter = self._getters.popleft()
            self.__put_internal(item)
            future_set_result_unless_cancelled(getter, self._get())
        elif self.full():
            raise QueueFull
        else:
            self.__put_internal(item)

    def get(self, timeout: Union[float, datetime.timedelta] = None) -> Awaitable[_T]:
        # 从队列中获取数据，如果队列为空则阻塞
        future = Future()  # type: Future[_T]
        try:
            future.set_result(self.get_nowait())
        except QueueEmpty:
            self._getters.append(future)
            _set_timeout(future, timeout)
        return future

    def get_nowait(self) -> _T:
        # 非阻塞从队列中获取数据
        self._consume_expired()
        if self._putters:
            assert self.full(), "queue not full, why are putters waiting?"
            item, putter = self._putters.popleft()
            self.__put_internal(item)
            future_set_result_unless_cancelled(putter, None)
            return self._get()
        elif self.qsize():
            return self._get()
        else:
            raise QueueEmpty

    def task_done(self) -> None:
        # 标识加入队列中的数据已经完成
        if self._unfinished_tasks <= 0:
            raise ValueError("task_done() called too many times")
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._finished.set()

    def join(self, timeout: Union[float, datetime.timedelta] = None) -> Awaitable[None]:
        # 阻塞直到队列中的所有项都被完成
        return self._finished.wait(timeout)

    def __aiter__(self) -> _QueueIterator[_T]:
        return _QueueIterator(self)

    # These three are overridable in subclasses.
    def _init(self) -> None:
        self._queue = collections.deque()

    def _get(self) -> _T:
        return self._queue.popleft()

    def _put(self, item: _T) -> None:
        self._queue.append(item)

    # End of the overridable methods.

    def __put_internal(self, item: _T) -> None:
        self._unfinished_tasks += 1
        self._finished.clear()
        self._put(item)

    def _consume_expired(self) -> None:
        # 删除超时的项
        while self._putters and self._putters[0][1].done():
            self._putters.popleft()

        while self._getters and self._getters[0].done():
            self._getters.popleft()
```

## class PriorityQueue
```python
class PriorityQueue(Queue):

    def _init(self) -> None:
        self._queue = []

    def _put(self, item: _T) -> None:
        heapq.heappush(self._queue, item)

    def _get(self) -> _T:
        return heapq.heappop(self._queue)
```
## class LifoQueue
```python
class LifoQueue(Queue):

    def _init(self) -> None:
        self._queue = []

    def _put(self, item: _T) -> None:
        self._queue.append(item)

    def _get(self) -> _T:
        return self._queue.pop()
```
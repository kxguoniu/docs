# tornado 之 gen.py

import asyncio
import builtins
import collections
from collections.abc import Generator
import concurrent.futures
import datetime
import functools
from functools import singledispatch
from inspect import isawaitable
import sys
import types

from tornado.concurrent import (
    Future,
    is_future,
    chain_future,
    future_set_exc_info,
    future_add_done_callback,
    future_set_result_unless_cancelled,
)
from tornado.ioloop import IOLoop
from tornado.log import app_log
from tornado.util import TimeoutError

import typing
from typing import Union, Any, Callable, List, Type, Tuple, Awaitable, Dict

if typing.TYPE_CHECKING:
    from typing import Sequence, Deque, Optional, Set, Iterable  # noqa: F401

_T = typing.TypeVar("_T")

_Yieldable = Union[
    None, Awaitable, List[Awaitable], Dict[Any, Awaitable], concurrent.futures.Future
]


class KeyReuseError(Exception):
    pass

class UnknownKeyError(Exception):
    pass

class LeakedCallbackError(Exception):
    pass

class BadYieldError(Exception):
    pass

class ReturnValueIgnoredError(Exception):
    pass

def _value_from_stopiteration(e: Union[StopIteration, "Return"]) -> Any:
    try:
        # StopIteration has a value attribute beginning in py33.
        # So does our Return class.
        return e.value
    except AttributeError:
        pass
    try:
        # Cython backports coroutine functionality by putting the value in
        # e.args[0].
        return e.args[0]
    except (AttributeError, IndexError):
        return None

## def _create_future
创建future
def _create_future() -> Future:
    future = Future()  # type: Future
    source_traceback = getattr(future, "_source_traceback", ())
    # 删除回溯当前文件
    while source_traceback:
        filename = source_traceback[-1][0]
        if filename == __file__:
            del source_traceback[-1]
        else:
            break
    return future


## def coroutine
协成装饰器
def coroutine(
    func: Callable[..., "Generator[Any, Any, _T]"]
) -> Callable[..., "Future[_T]"]:
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        future = _create_future()
        try:
            result = func(*args, **kwargs)
        except (Return, StopIteration) as e:
            result = _value_from_stopiteration(e)
        except Exception:
            future_set_exc_info(future, sys.exc_info())
            try:
                return future
            finally:
                # Avoid circular references
                future = None  # type: ignore
        else:
            if isinstance(result, Generator):
                try:
                    yielded = next(result)
                except (StopIteration, Return) as e:
                    future_set_result_unless_cancelled(
                        future, _value_from_stopiteration(e)
                    )
                except Exception:
                    future_set_exc_info(future, sys.exc_info())
                else:
                    runner = Runner(result, future, yielded)
                    future.add_done_callback(lambda _: runner)
                yielded = None
                try:
                    return future
                finally:
                    future = None  # type: ignore
        future_set_result_unless_cancelled(future, result)
        return future

    wrapper.__wrapped__ = func  # type: ignore
    wrapper.__tornado_coroutine__ = True  # type: ignore
    return wrapper

## def is_coroutine_function
判断对象是不是携程(gen.coroutine)
def is_coroutine_function(func: Any) -> bool:
    return getattr(func, "__tornado_coroutine__", False)

## 协程异常返回，3.3以上不再需要
class Return(Exception):
    def __init__(self, value: Any = None) -> None:
        super(Return, self).__init__()
        self.value = value
        self.args = (value,)

## 协程的迭代器，先完成的先返回
class WaitIterator(object):
    _unfinished = {}  # type: Dict[Future, Union[int, str]]

    def __init__(self, *args: Future, **kwargs: Future) -> None:
        if args and kwargs:
            raise ValueError("You must provide args or kwargs, not both")

        if kwargs:
            self._unfinished = dict((f, k) for (k, f) in kwargs.items())
            futures = list(kwargs.values())  # type: Sequence[Future]
        else:
            self._unfinished = dict((f, i) for (i, f) in enumerate(args))
            futures = args

        self._finished = collections.deque()  # type: Deque[Future]
        self.current_index = None  # type: Optional[Union[str, int]]
        self.current_future = None  # type: Optional[Future]
        self._running_future = None  # type: Optional[Future]

        for future in futures:
            future_add_done_callback(future, self._done_callback)

    # 迭代器结束
    def done(self) -> bool:
        """Returns True if this iterator has no more results."""
        if self._finished or self._unfinished:
            return False
        self.current_index = self.current_future = None
        return True

    # 获取下一个future结果
    def next(self) -> Future:
        self._running_future = Future()

        if self._finished:
            self._return_result(self._finished.popleft())

        return self._running_future

    # 协程完成回调函数
    def _done_callback(self, done: Future) -> None:
        if self._running_future and not self._running_future.done():
            self._return_result(done)
        else:
            self._finished.append(done)

    # 返回结果
    def _return_result(self, done: Future) -> None:
        if self._running_future is None:
            raise Exception("no future is running")
        chain_future(done, self._running_future)

        self.current_future = done
        self.current_index = self._unfinished.pop(done)

    # 异步迭代器
    def __aiter__(self) -> typing.AsyncIterator:
        return self

    def __anext__(self) -> Future:
        if self.done():
            # Lookup by name to silence pyflakes on older versions.
            raise getattr(builtins, "StopAsyncIteration")()
        return self.next()

## 并行运行多个异步程序
def multi(
    children: Union[List[_Yieldable], Dict[Any, _Yieldable]],
    quiet_exceptions: "Union[Type[Exception], Tuple[Type[Exception], ...]]" = (),
) -> "Union[Future[List], Future[Dict]]":
    return multi_future(children, quiet_exceptions=quiet_exceptions)


Multi = multi


def multi_future(
    children: Union[List[_Yieldable], Dict[Any, _Yieldable]],
    quiet_exceptions: "Union[Type[Exception], Tuple[Type[Exception], ...]]" = (),
) -> "Union[Future[List], Future[Dict]]":
    if isinstance(children, dict):
        keys = list(children.keys())  # type: Optional[List]
        children_seq = children.values()  # type: Iterable
    else:
        keys = None
        children_seq = children
    children_futs = list(map(convert_yielded, children_seq))
    assert all(is_future(i) or isinstance(i, _NullFuture) for i in children_futs)
    unfinished_children = set(children_futs)

    future = _create_future()
    if not children_futs:
        future_set_result_unless_cancelled(future, {} if keys is not None else [])

    def callback(fut: Future) -> None:
        unfinished_children.remove(fut)
        if not unfinished_children:
            result_list = []
            for f in children_futs:
                try:
                    result_list.append(f.result())
                except Exception as e:
                    if future.done():
                        if not isinstance(e, quiet_exceptions):
                            app_log.error(
                                "Multiple exceptions in yield list", exc_info=True
                            )
                    else:
                        future_set_exc_info(future, sys.exc_info())
            if not future.done():
                if keys is not None:
                    future_set_result_unless_cancelled(
                        future, dict(zip(keys, result_list))
                    )
                else:
                    future_set_result_unless_cancelled(future, result_list)

    listening = set()  # type: Set[Future]
    for f in children_futs:
        if f not in listening:
            listening.add(f)
            future_add_done_callback(f, callback)
    return future


def maybe_future(x: Any) -> Future:
    if is_future(x):
        return x
    else:
        fut = _create_future()
        fut.set_result(x)
        return fut


def with_timeout(
    timeout: Union[float, datetime.timedelta],
    future: _Yieldable,
    quiet_exceptions: "Union[Type[Exception], Tuple[Type[Exception], ...]]" = (),
) -> Future:
    # It's tempting to optimize this by cancelling the input future on timeout
    # instead of creating a new one, but A) we can't know if we are the only
    # one waiting on the input future, so cancelling it might disrupt other
    # callers and B) concurrent futures can only be cancelled while they are
    # in the queue, so cancellation cannot reliably bound our waiting time.
    future_converted = convert_yielded(future)
    result = _create_future()
    chain_future(future_converted, result)
    io_loop = IOLoop.current()

    def error_callback(future: Future) -> None:
        try:
            future.result()
        except asyncio.CancelledError:
            pass
        except Exception as e:
            if not isinstance(e, quiet_exceptions):
                app_log.error(
                    "Exception in Future %r after timeout", future, exc_info=True
                )

    def timeout_callback() -> None:
        if not result.done():
            result.set_exception(TimeoutError("Timeout"))
        # In case the wrapped future goes on to fail, log it.
        future_add_done_callback(future_converted, error_callback)

    timeout_handle = io_loop.add_timeout(timeout, timeout_callback)
    if isinstance(future_converted, Future):
        # We know this future will resolve on the IOLoop, so we don't
        # need the extra thread-safety of IOLoop.add_future (and we also
        # don't care about StackContext here.
        future_add_done_callback(
            future_converted, lambda future: io_loop.remove_timeout(timeout_handle)
        )
    else:
        # concurrent.futures.Futures may resolve on any thread, so we
        # need to route them back to the IOLoop.
        io_loop.add_future(
            future_converted, lambda future: io_loop.remove_timeout(timeout_handle)
        )
    return result


def sleep(duration: float) -> "Future[None]":
    f = _create_future()
    IOLoop.current().call_later(
        duration, lambda: future_set_result_unless_cancelled(f, None)
    )
    return f


class _NullFuture(object):
    def result(self) -> None:
        return None

    def done(self) -> bool:
        return True


# _null_future is used as a dummy value in the coroutine runner. It differs
# from moment in that moment always adds a delay of one IOLoop iteration
# while _null_future is processed as soon as possible.
_null_future = typing.cast(Future, _NullFuture())

moment = typing.cast(Future, _NullFuture())
moment.__doc__ = """A special object which may be yielded to allow the IOLoop to run for one iteration."""

# 协程包装器的内部实现
class Runner(object):
    def __init__(
        self,
        gen: "Generator[_Yieldable, Any, _T]",
        result_future: "Future[_T]",
        first_yielded: _Yieldable,
    ) -> None:
        self.gen = gen
        self.result_future = result_future
        self.future = _null_future  # type: Union[None, Future]
        self.running = False
        self.finished = False
        self.io_loop = IOLoop.current()
        if self.handle_yield(first_yielded):
            gen = result_future = first_yielded = None  # type: ignore
            self.run()

    def run(self) -> None:
        """Starts or resumes the generator, running until it reaches a
        yield point that is not ready.
        """
        if self.running or self.finished:
            return
        try:
            self.running = True
            while True:
                future = self.future
                if future is None:
                    raise Exception("No pending future")
                if not future.done():
                    return
                self.future = None
                try:
                    exc_info = None

                    try:
                        value = future.result()
                    except Exception:
                        exc_info = sys.exc_info()
                    future = None

                    if exc_info is not None:
                        try:
                            yielded = self.gen.throw(*exc_info)  # type: ignore
                        finally:
                            # Break up a reference to itself
                            # for faster GC on CPython.
                            exc_info = None
                    else:
                        yielded = self.gen.send(value)

                except (StopIteration, Return) as e:
                    self.finished = True
                    self.future = _null_future
                    future_set_result_unless_cancelled(
                        self.result_future, _value_from_stopiteration(e)
                    )
                    self.result_future = None  # type: ignore
                    return
                except Exception:
                    self.finished = True
                    self.future = _null_future
                    future_set_exc_info(self.result_future, sys.exc_info())
                    self.result_future = None  # type: ignore
                    return
                if not self.handle_yield(yielded):
                    return
                yielded = None
        finally:
            self.running = False

    def handle_yield(self, yielded: _Yieldable) -> bool:
        try:
            self.future = convert_yielded(yielded)
        except BadYieldError:
            self.future = Future()
            future_set_exc_info(self.future, sys.exc_info())

        if self.future is moment:
            self.io_loop.add_callback(self.run)
            return False
        elif self.future is None:
            raise Exception("no pending future")
        elif not self.future.done():

            def inner(f: Any) -> None:
                # Break a reference cycle to speed GC.
                f = None  # noqa: F841
                self.run()

            self.io_loop.add_future(self.future, inner)
            return False
        return True

    def handle_exception(
        self, typ: Type[Exception], value: Exception, tb: types.TracebackType
    ) -> bool:
        if not self.running and not self.finished:
            self.future = Future()
            future_set_exc_info(self.future, (typ, value, tb))
            self.run()
            return True
        else:
            return False


# Convert Awaitables into Futures.
try:
    _wrap_awaitable = asyncio.ensure_future
except AttributeError:
    # asyncio.ensure_future was introduced in Python 3.4.4, but
    # Debian jessie still ships with 3.4.2 so try the old name.
    _wrap_awaitable = getattr(asyncio, "async")


def convert_yielded(yielded: _Yieldable) -> Future:

    if yielded is None or yielded is moment:
        return moment
    elif yielded is _null_future:
        return _null_future
    elif isinstance(yielded, (list, dict)):
        return multi(yielded)  # type: ignore
    elif is_future(yielded):
        return typing.cast(Future, yielded)
    elif isawaitable(yielded):
        return _wrap_awaitable(yielded)  # type: ignore
    else:
        raise BadYieldError("yielded unknown object %r" % (yielded,))


convert_yielded = singledispatch(convert_yielded)

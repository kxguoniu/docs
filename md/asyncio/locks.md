# asyncio 之 locks.py
## 原语锁 Lock
### 初始化
```python
class Lock:
    def __init__(self, *, loop=None):
		# 等待获取锁的对象队列
        self._waiters = collections.deque()
		# 锁是否被占用
        self._locked = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()

    def locked(self):
        # 如果锁被占用返回 True
        return self._locked

	# 获取锁
    @coroutine
    def acquire(self):
        # 如果没有其他等待获取锁的对象并且锁状态为假
        if not self._waiters and not self._locked:
			# 设置锁状态为真并返回
            self._locked = True
            return True
		# 创建一个等待获取锁 future
        fut = futures.Future(loop=self._loop)
		# 添加到等待队列中
        self._waiters.append(fut)
		# 尝试获取锁，获取到之后把future从等待队列中删除
        try:
            yield from fut
            self._locked = True
            return True
        finally:
            self._waiters.remove(fut)
	# 释放锁
    def release(self):
        if self._locked:
			# 设置锁状态为假
            self._locked = False
            # 给没有取消的第一个等待获取锁的 future 设置结果
            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(True)
                    break
        else:
            raise RuntimeError('Lock is not acquired.')
	# 不允许直接使用上下文管理器进入
    def __enter__(self):
        raise RuntimeError(
            '"yield from" should be used as context manager expression')

    def __exit__(self, *args):
        pass
	# 上下文管理起从这里进入  with (yield from lock)
    def __iter__(self):
        yield from self.acquire()
		# 返回一个上下文管理器处理进入和退出时的操作
        return _ContextManager(self)
```
## 上下文管理器
### pass
```python
class _ContextManager:
    def __init__(self, lock):
        self._lock = lock
	# 进入时什么都不做
    def __enter__(self):
        return None
	# 退出时释放锁
    def __exit__(self, *args):
        try:
            self._lock.release()
        finally:
            self._lock = None
```
## Event 锁
### 初始化
```python
class Event:
    def __init__(self, *, loop=None):
		# 等待 event 状态为真的 future 对象队列
        self._waiters = collections.deque()
		# event 的状态
        self._value = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
	# event 是不是在被占用
    def is_set(self):
        return self._value
	# 将event状态设置为真，所有等待event为真的future对象将被设置结果
    def set(self):
        if not self._value:
            self._value = True

            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(True)
	# 将event状态设置为False，所有调用 wait() 的协程将被阻塞
    def clear(self):
        self._value = False
	# 阻塞直到标识为真
    @coroutine
    def wait(self):
        if self._value:
            return True

        fut = futures.Future(loop=self._loop)
        self._waiters.append(fut)
        try:
            yield from fut
            return True
        finally:
            self._waiters.remove(fut)
```
## 条件锁
```python
class Condition:
    def __init__(self, lock=None, *, loop=None):
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()

        if lock is None:
            lock = Lock(loop=self._loop)
        elif lock._loop is not self._loop:
            raise ValueError("loop argument must agree with lock")

        self._lock = lock
        # 导出锁的三种方法
        self.locked = lock.locked
        self.acquire = lock.acquire
        self.release = lock.release
		# 等待队列
        self._waiters = collections.deque()

    @coroutine
    def wait(self):
        """Wait until notified.

        If the calling coroutine has not acquired the lock when this
        method is called, a RuntimeError is raised.

        This method releases the underlying lock, and then blocks
        until it is awakened by a notify() or notify_all() call for
        the same condition variable in another coroutine.  Once
        awakened, it re-acquires the lock and returns True.
        """
        if not self.locked():
            raise RuntimeError('cannot wait on un-acquired lock')

        self.release()
        try:
            fut = futures.Future(loop=self._loop)
            self._waiters.append(fut)
            try:
                yield from fut
                return True
            finally:
                self._waiters.remove(fut)

        finally:
            yield from self.acquire()

    @coroutine
    def wait_for(self, predicate):
        """Wait until a predicate becomes true.

        The predicate should be a callable which result will be
        interpreted as a boolean value.  The final predicate value is
        the return value.
        """
        result = predicate()
        while not result:
            yield from self.wait()
            result = predicate()
        return result

    def notify(self, n=1):
        """By default, wake up one coroutine waiting on this condition, if any.
        If the calling coroutine has not acquired the lock when this method
        is called, a RuntimeError is raised.

        This method wakes up at most n of the coroutines waiting for the
        condition variable; it is a no-op if no coroutines are waiting.

        Note: an awakened coroutine does not actually return from its
        wait() call until it can reacquire the lock. Since notify() does
        not release the lock, its caller should.
        """
        if not self.locked():
            raise RuntimeError('cannot notify on un-acquired lock')

        idx = 0
        for fut in self._waiters:
            if idx >= n:
                break

            if not fut.done():
                idx += 1
                fut.set_result(False)

    def notify_all(self):
        """Wake up all threads waiting on this condition. This method acts
        like notify(), but wakes up all waiting threads instead of one. If the
        calling thread has not acquired the lock when this method is called,
        a RuntimeError is raised.
        """
        self.notify(len(self._waiters))

    def __enter__(self):
        raise RuntimeError(
            '"yield from" should be used as context manager expression')

    def __exit__(self, *args):
        pass

    def __iter__(self):
        # See comment in Lock.__iter__().
        yield from self.acquire()
        return _ContextManager(self)
```
### 信号锁
```python
class Semaphore:
    def __init__(self, value=1, *, loop=None):
        if value < 0:
            raise ValueError("Semaphore initial value must be >= 0")
        self._value = value
        self._waiters = collections.deque()
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
	# 如果锁的数量为0则返回真
    def locked(self):
        return self._value == 0

    @coroutine
    def acquire(self):
        if not self._waiters and self._value > 0:
            self._value -= 1
            return True

        fut = futures.Future(loop=self._loop)
        self._waiters.append(fut)
        try:
            yield from fut
            self._value -= 1
            return True
        finally:
            self._waiters.remove(fut)

    def release(self):
        self._value += 1
        for waiter in self._waiters:
            if not waiter.done():
                waiter.set_result(True)
                break

    def __enter__(self):
        raise RuntimeError(
            '"yield from" should be used as context manager expression')

    def __exit__(self, *args):
        pass

    def __iter__(self):
        # See comment in Lock.__iter__().
        yield from self.acquire()
        return _ContextManager(self)
```
## 有界信号量
```python
class BoundedSemaphore(Semaphore):
    def __init__(self, value=1, *, loop=None):
        self._bound_value = value
        super().__init__(value, loop=loop)

    def release(self):
        if self._value >= self._bound_value:
            raise ValueError('BoundedSemaphore released too many times')
        super().release()
```
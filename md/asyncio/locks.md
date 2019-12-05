[TOC]
## 摘要
`locks.py`中定义了`Lock`、`Event`、`Condition`、`Semaphore`、`BoundeSemaphore`的异步实现。
## class _ContextManager
进入的时候什么都不做，退出的时候释放锁。
```python
class _ContextManager:
    """
        with (yield from lock):
            <block>

        async with lock:
            <block>
    """
    def __init__(self, lock):
        self._lock = lock
    def __enter__(self):
        return None
    def __exit__(self, *args):
        try:
            self._lock.release()
        finally:
            self._lock = None  # Crudely prevent reuse.
```
## class _ContextManagerMixin
```python
class _ContextManagerMixin:
    def __enter__(self):
        raise RuntimeError(
            '"yield from" should be used as context manager expression')

	# 不允许直接进入，所以退出用不到
    def __exit__(self, *args):
        pass

	# 支持  with (yield from lock)
    @coroutine
    def __iter__(self):
        warnings.warn("'with (yield from lock)' is deprecated "
                      "use 'async with lock' instead",
                      DeprecationWarning, stacklevel=2)
        yield from self.acquire()
        return _ContextManager(self)

    async def __acquire_ctx(self):
        await self.acquire()
        return _ContextManager(self)

	# 支持 with await lock
    def __await__(self):
        warnings.warn("'with await lock' is deprecated "
                      "use 'async with lock' instead",
                      DeprecationWarning, stacklevel=2)
        # To make "with await lock" work.
        return self.__acquire_ctx().__await__()

	# 支持 async with lock
    async def __aenter__(self):
        await self.acquire()
        return None

    async def __aexit__(self, exc_type, exc, tb):
        self.release()
```
## class Lock
原语锁，一次只允许一个协程获取锁并执行。
### 初始化
```python
class Lock(_ContextManagerMixin):
    def __init__(self, *, loop=None):
		# 等待获取锁的协程的 future 队列
        self._waiters = collections.deque()
		# 锁状态
        self._locked = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
```
### 获取锁状态
```python
def locked(self):
	return self._locked
```
### 获取锁
```python
async def acquire(self):
	# 锁可用并且没有等待获取锁的 future
	if not self._locked and all(w.cancelled() for w in self._waiters):
		self._locked = True
		return True

	# 创建 future 添加到等待队列中
	fut = self._loop.create_future()
	self._waiters.append(fut)
	# 阻塞协程直到获取锁成功并把 future 从队列中删除
	try:
		try:
			await fut
		finally:
			self._waiters.remove(fut)
	# 如果 fut 被取消，唤醒队列中的下一个并抛出异常
	except futures.CancelledError:
		if not self._locked:
			self._wake_up_first()
		raise

	self._locked = True
	return True
```
### 释放锁
```python
def release(self):
	# 释放锁并唤醒等待队列中的第一个
	if self._locked:
		self._locked = False
		self._wake_up_first()
	else:
		raise RuntimeError('Lock is not acquired.')
```
### def _wake_up_first
```python
def _wake_up_first(self):
	# 取出队列中的第一个，如果队列为空，捕获异常什么也不做
	try:
		fut = next(iter(self._waiters))
	except StopIteration:
		return
	# 给取出的第一个 fut 设置结果。
	if not fut.done():
		fut.set_result(True)
```
## class Event
事件锁，当一个协程调用`clear`方法后所有之后调用`wait`方法的协程都会被阻塞直到这个协程完成并调用`set`方法将它们唤醒继续执行。
```python
class Event:
    def __init__(self, *, loop=None):
		# 被阻塞协程的 future 队列
        self._waiters = collections.deque()
		# 锁状态
        self._value = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()

	# 查看锁状态
    def is_set(self):
        return self._value

	# 将状态设置为真，唤醒所有阻塞的程序并且在之后调用 wait 方法的程序也不会阻塞
    def set(self):
        if not self._value:
            self._value = True

            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(True)

	# 重置event状态，之后所有调用wait方法的程序都会被阻塞
    def clear(self):
        self._value = False

	# 如果状态不为真，阻塞程序直到其他程序调用 set 方法。
    async def wait(self):
        if self._value:
            return True

		# 创建一个future添加到队列中并阻塞。等待被唤醒。
        fut = self._loop.create_future()
        self._waiters.append(fut)
        try:
            await fut
            return True
        finally:
            self._waiters.remove(fut)
```
## class Condition
Condition(条件变量)，所有协程调用`wait`方法后都会进入阻塞状态，直到另一个协程调用`notify/notify_all`方法通知，得到通知的协程去尝试获取锁。`Condition`通常与一个锁关联，需要在多个`Contidion`中共享一个锁时，可以传递一个`lock`对象给构造方法，否则会自己创建一个`Lock`实例。
### 初始化
```python
class Condition(_ContextManagerMixin):
    def __init__(self, lock=None, *, loop=None):
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
        if lock is None:
            lock = Lock(loop=self._loop)
        elif lock._loop is not self._loop:
            raise ValueError("loop argument must agree with lock")

		# 内部锁以及锁的方法
        self._lock = lock
        self.locked = lock.locked
        self.acquire = lock.acquire
        self.release = lock.release
		# 被阻塞协程的 future 队列
        self._waiters = collections.deque()
```
### 阻塞
协程调用`wait`方法之前先获得锁，进入`wait`方法后释放锁并创建`future`阻塞协程，等到`future`完成之后在退出`wait`之前再次获取锁，退出之后再次释放锁。
在协程阻塞之前释放锁是为了其他协程也可以获得锁，退出之前获取锁则与退出方法后的释放锁对应。
```python
async def wait(self):
	if not self.locked():
		raise RuntimeError('cannot wait on un-acquired lock')
	# 先释放锁，让其他协程可以进入
	self.release()
	try:
		fut = self._loop.create_future()
		self._waiters.append(fut)
		try:
			await fut
			return True
		finally:
			self._waiters.remove(fut)
	# 任务完成之后必须要获取锁，才能退出。
	finally:
		cancelled = False
		while True:
			try:
				await self.acquire()
				break
			except futures.CancelledError:
				cancelled = True
		# 如果等待被取消，抛出异常。
		if cancelled:
			raise futures.CancelledError
```
### 阻塞协程直到可调用对象返回的结果为真
```
async def wait_for(self, predicate):
	result = predicate()
	while not result:
		await self.wait()
		result = predicate()
	return result
```
### 唤醒
协程调用`notify`方法唤醒其他被阻塞的协程，默认参数`n`表示最大的唤醒个数，调用`notify_all`则会唤醒所有被阻塞的协程。
```python
def notify(self, n=1):
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
	self.notify(len(self._waiters))
```
## class Semaphore
### 初始化
```python
class Semaphore(_ContextManagerMixin):
    def __init__(self, value=1, *, loop=None):
        if value < 0:
            raise ValueError("Semaphore initial value must be >= 0")
        # 锁的数量
		self._value = value
		# 被阻塞协程的 future 队列
        self._waiters = collections.deque()
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
```
### 状态和内置方法
```python
def _wake_up_next(self):
	while self._waiters:
		waiter = self._waiters.popleft()
		if not waiter.done():
			waiter.set_result(None)
			return

def locked(self):
	return self._value == 0
```
### 获取/释放锁
- 协程每次获取锁都会让锁的数量减一，当锁的数量为0时协程会被阻塞。
- 协程每次释放锁都会让锁的数量加一，并尝试唤醒等待队列里的第一个被阻塞的协程。
```python
async def acquire(self):
	while self._value <= 0:
		fut = self._loop.create_future()
		self._waiters.append(fut)
		try:
			await fut
		except:
			# See the similar code in Queue.get.
			fut.cancel()
			if self._value > 0 and not fut.cancelled():
				self._wake_up_next()
			raise
	self._value -= 1
	return True

def release(self):
	self._value += 1
	self._wake_up_next()
```
## class BoundedSemaphore
限定了内部锁的数量不能超过初始化的默认值。也就是释放锁的次数不能大于获取锁的次数
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
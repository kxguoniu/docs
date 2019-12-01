# asyncio 之 locks.py
## 上下文管理器
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
## Lock
### 初始化
```python
class Lock(_ContextManagerMixin):
    def __init__(self, *, loop=None):
        self._waiters = collections.deque()
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
@coroutine
def acquire(self):
	# 如果等待队列为空并且锁可用
	if not self._waiters and not self._locked:
		self._locked = True
		return True
	# 创建 future
	fut = futures.Future(loop=self._loop)
	# 把 future 添加到等待队列中
	self._waiters.append(fut)
	# 阻塞直到 future 完成，然后把 future 从等待队列中删除
	try:
		yield from fut
		self._locked = True
		return True
	finally:
		self._waiters.remove(fut)

async def acquire(self):
	# 锁可用并且没有等待获取锁的 future
	if not self._locked and all(w.cancelled() for w in self._waiters):
		self._locked = True
		return True

	# 创建 future 添加到等待队列中
	fut = self._loop.create_future()
	self._waiters.append(fut)

	try:
		# 阻塞程序直到获取锁成功并把 future 从队列中删除
		try:
			await fut
		finally:
			self._waiters.remove(fut)
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
	if self._locked:
		self._locked = False
		self._wake_up_first()
	else:
		raise RuntimeError('Lock is not acquired.')
```
### def _wake_up_first
```python
def _wake_up_first(self):
	try:
		fut = next(iter(self._waiters))
	except StopIteration:
		return

	if not fut.done():
		fut.set_result(True)
```
## Event
```python
class Event:
    def __init__(self, *, loop=None):
        # 被阻塞程序的 future 队列
        self._waiters = collections.deque()
        # 状态
        self._value = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()

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

        fut = self._loop.create_future()
        self._waiters.append(fut)
        try:
            await fut
            return True
        finally:
            self._waiters.remove(fut)

```
## Condition
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
		# 阻塞程序的 future 队列
        self._waiters = collections.deque()
```
### 阻塞
协程调用`wait`方法之前先获得锁，进入`wait`方法后释放锁并创建`future`阻塞协程，等到`future`完成之后在退出`wait`之前再次获取锁，退出之后再次释放锁。
在协程阻塞之前释放锁是为了其他协程也可以获得锁，退出之前获取锁则与退出方法后的释放锁对应。
```python
async def wait(self):
	if not self.locked():
		raise RuntimeError('cannot wait on un-acquired lock')

	self.release()
	try:
		fut = self._loop.create_future()
		self._waiters.append(fut)
		try:
			await fut
			return True
		finally:
			self._waiters.remove(fut)

	finally:
		cancelled = False
		while True:
			try:
				await self.acquire()
				break
			except futures.CancelledError:
				cancelled = True

		if cancelled:
			raise futures.CancelledError
```
# 阻塞协程直到可调用对象返回的结果为真
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
## Semaphore
### 初始化
```python
class Semaphore:
    def __init__(self, value=1, *, loop=None):
        if value < 0:
            raise ValueError("Semaphore initial value must be >= 0")
		# 锁的数量
        self._value = value
		# 阻塞的协程的 future 队列
        self._waiters = collections.deque()
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
```
### 状态和内置方法
```python
def locked(self):
	return self._value == 0

def _wake_up_next(self):
	while self._waiters:
		waiter = self._waiters.popleft()
		if not waiter.done():
			waiter.set_result(None)
			return
			
```
### 获取/释放锁
协程每次获取锁都会让锁的数量减一，当锁的数量为0时协程会被阻塞。
协程每次释放锁都会让锁的数量加一，并尝试唤醒等待队列里的第一个被阻塞的协程。
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
## BoundedSemaphore
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
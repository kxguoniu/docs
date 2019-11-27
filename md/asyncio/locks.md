# asyncio 之 locks.py
## 上下文管理器
进入的时候什么都不做，退出的时候释放锁。
```python
class _ContextManager:
    def __init__(self, lock):
        self._lock = lock
    def __enter__(self):
        return None
    def __exit__(self, *args):
        try:
            self._lock.release()
        finally:
            self._lock = None
```
## Lock
### 初始化
```python
class Lock:
    def __init__(self, *, loop=None):
		# 等待获取锁的 future 队列
        self._waiters = collections.deque()
		# 锁状态
        self._locked = False
        if loop is not None:
            self._loop = loop
        else:
            self._loop = events.get_event_loop()
```
### 内置方法
```python
# 如果锁被占用返回 True
def locked(self):
	return self._locked

# 不允许直接使用上下文管理器进入
def __enter__(self):
	raise RuntimeError(
		'"yield from" should be used as context manager expression')
def __exit__(self, *args):
	pass

# 上下文管理器从这里进入  with (yield from lock)
def __iter__(self):
	yield from self.acquire()
	return _ContextManager(self)
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
```
### 释放锁
```python
def release(self):
	if self._locked:
		self._locked = False
		# 给等待队列中的第一个未取消的 future 设置结果
		for fut in self._waiters:
			if not fut.done():
				fut.set_result(True)
				break
	else:
		raise RuntimeError('Lock is not acquired.')
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

	# 将event状态设置为真，唤醒所有阻塞的程序并且在之后调用 wait 方法的程序也不会被阻塞。
    def set(self):
        if not self._value:
            self._value = True
            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(True)

	# 重置 event 的状态，所有调用 wait 方法的程序都会被阻塞
    def clear(self):
        self._value = False

	# 如果 event 状态为False，阻塞程序直到其他程序调用 set 方法
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
## Condition
### 初始化
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

		# 内部锁以及锁的方法
        self._lock = lock
        self.locked = lock.locked
        self.acquire = lock.acquire
        self.release = lock.release
		# 阻塞协程的 future 队列
        self._waiters = collections.deque()
```
### 阻塞
协程调用`wait`方法之前先获得锁，进入`wait`方法后释放锁并创建`future`阻塞协程，等到`future`完成之后在退出`wait`之前再次获取锁，退出之后再次释放锁。
在协程阻塞之前释放锁是为了其他协程也可以获得锁，退出之前获取锁则与退出方法后的释放锁对应。
```python
@coroutine
def wait(self):
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

# 阻塞协程直到可调用对象返回的结果为真
@coroutine
def wait_for(self, predicate):
	result = predicate()
	while not result:
		yield from self.wait()
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
### 上下文
```python
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
### Semaphore
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
	# 如果锁的数量为0则返回真
    def locked(self):
        return self._value == 0
```
### 获取/释放锁
协程每次获取锁都会让锁的数量减一，当锁的数量为0时协程会被阻塞。
协程每次释放锁都会让锁的数量加一，并尝试唤醒等待队列里的第一个被阻塞的协程。
```python
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
```
### 上下文
```python
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
# asyncio 之 queues.py
## Queue
### 初始化队列
```python
def __init__(self, maxsize=0, *, loop=None):
	# 设置 loop
	if loop is None:
		self._loop = events.get_event_loop()
	else:
		self._loop = loop
	# 队列最大长度
	self._maxsize = maxsize

	# Futures.
	self._getters = collections.deque()
	# Pairs of (item, Future).
	self._putters = collections.deque()
	self._init(maxsize)

def _init(self, maxsize):
	self._queue = collections.deque()
```
### 内置的 get 和 put
```python
def _get(self):
	return self._queue.popleft()

def _put(self, item):
	self._queue.append(item)
```
### TODO
```python
def _consume_done_getters(self):
	# Delete waiters at the head of the get() queue who've timed out.
	while self._getters and self._getters[0].done():
		self._getters.popleft()

def _consume_done_putters(self):
	# Delete waiters at the head of the put() queue who've timed out.
	while self._putters and self._putters[0][1].done():
		self._putters.popleft()
```
### 获取队列的状态的函数
```python
def qsize(self):
	"""Number of items in the queue."""
	return len(self._queue)

@property
def maxsize(self):
	"""Number of items allowed in the queue."""
	return self._maxsize

def empty(self):
	"""Return True if the queue is empty, False otherwise."""
	return not self._queue

def full(self):
	"""Return True if there are maxsize items in the queue.

	Note: if the Queue was initialized with maxsize=0 (the default),
	then full() is never True.
	"""
	if self._maxsize <= 0:
		return False
	else:
		return self.qsize() >= self._maxsize
```
### 向队列添加数据
```python
@coroutine
def put(self, item):
	"""Put an item into the queue.

	Put an item into the queue. If the queue is full, wait until a free
	slot is available before adding item.

	This method is a coroutine.
	"""
	self._consume_done_getters()
	if self._getters:
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')

		getter = self._getters.popleft()

		# Use _put and _get instead of passing item straight to getter, in
		# case a subclass has logic that must run (e.g. JoinableQueue).
		self._put(item)

		# getter cannot be cancelled, we just removed done getters
		getter.set_result(self._get())

	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		waiter = futures.Future(loop=self._loop)

		self._putters.append((item, waiter))
		yield from waiter

	else:
		self._put(item)
```
### 非阻塞向队列添加数据
```python
def put_nowait(self, item):
	"""Put an item into the queue without blocking.

	If no free slot is immediately available, raise QueueFull.
	"""
	self._consume_done_getters()
	if self._getters:
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')

		getter = self._getters.popleft()

		# Use _put and _get instead of passing item straight to getter, in
		# case a subclass has logic that must run (e.g. JoinableQueue).
		self._put(item)

		# getter cannot be cancelled, we just removed done getters
		getter.set_result(self._get())

	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		raise QueueFull
	else:
		self._put(item)
```
### 从队列获取数据
```python
@coroutine
def get(self):
	"""Remove and return an item from the queue.

	If queue is empty, wait until an item is available.

	This method is a coroutine.
	"""
	self._consume_done_putters()
	if self._putters:
		assert self.full(), 'queue not full, why are putters waiting?'
		item, putter = self._putters.popleft()
		self._put(item)

		# When a getter runs and frees up a slot so this putter can
		# run, we need to defer the put for a tick to ensure that
		# getters and putters alternate perfectly. See
		# ChannelTest.test_wait.
		self._loop.call_soon(putter._set_result_unless_cancelled, None)

		return self._get()

	elif self.qsize():
		return self._get()
	else:
		waiter = futures.Future(loop=self._loop)

		self._getters.append(waiter)
		return (yield from waiter)
```
### 非阻塞
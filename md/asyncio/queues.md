[TOC]
# asyncio 之 queues.py
## class Queue
### 初始化队列
队列初始化的时候会创建`getters`队列、`putters`队列和`queue`数据队列
当`queue`为空时，所有的`get`操作都会放在`getters`队列中等待`put`操作。
当`queue`为满时，所有的`put`操作都会放在`putters`队列中等待`get`操作。
```python
def __init__(self, maxsize=0, *, loop=None):
	if loop is None:
		self._loop = events.get_event_loop()
	else:
		self._loop = loop
	# 队列的最大长度
	self._maxsize = maxsize

	# Futures.
	self._getters = collections.deque()
	# Futures.
	self._putters = collections.deque()
	# 未完成的任务数量
	self._unfinished_tasks = 0
	# 事件锁
	self._finished = locks.Event(loop=self._loop)
	# 阻塞
	self._finished.set()
	self._init(maxsize)
```
### 队列内置方法
前三个方法在子类中是可以覆盖的。
```python
def _init(self, maxsize):
	self._queue = collections.deque()

def _get(self):
	return self._queue.popleft()

def _put(self, item):
	self._queue.append(item)

# 唤醒等待队列中第一个未取消的任务
def _wakeup_next(self, waiters):
	while waiters:
		waiter = waiters.popleft()
		if not waiter.done():
			waiter.set_result(None)
			break
```
### 队列状态获取方法
```python
def qsize(self):
	return len(self._queue)

@property
def maxsize(self):
	return self._maxsize

def empty(self):
	return not self._queue

def full(self):
	if self._maxsize <= 0:
		return False
	else:
		return self.qsize() >= self._maxsize
```
### def put
向队列添加数据，如果队列已满，则阻塞到添加完成。
```python
async def put(self, item):
	# 如果队列为满则需阻塞
	while self.full():
		# 向队列中添加一个 put future
		putter = self._loop.create_future()
		self._putters.append(putter)
		# 阻塞
		try:
			await putter
		except:
			putter.cancel()  # Just in case putter is not done yet.
			try:
				self._putters.remove(putter)
			except ValueError:
				pass
			if not self.full() and not putter.cancelled():
				# We were woken up by get_nowait(), but can't take
				# the call.  Wake up the next in line.
				self._wakeup_next(self._putters)
			raise
	# 直到队列不满，向其添加数据
	return self.put_nowait(item)
```
### def put_nowait
非阻塞向队列中添加数据，如果队列已满，抛出异常。
```python
def put_nowait(self, item):
	if self.full():
		raise QueueFull
	self._put(item)
	self._unfinished_tasks += 1
	self._finished.clear()
	# 尝试唤醒一个被阻塞的获取队列数据的程序
	self._wakeup_next(self._getters)
```
### def get
从队列获取数据，如果队列为空，阻塞直到队列有数据。
```python
async def get(self):
	while self.empty():
		# 向队列中添加一个 get future
		getter = self._loop.create_future()
		self._getters.append(getter)
		# 阻塞
		try:
			await getter
		except:
			getter.cancel()  # Just in case getter is not done yet.
			try:
				self._getters.remove(getter)
			except ValueError:
				pass
			if not self.empty() and not getter.cancelled():
				self._wakeup_next(self._getters)
			raise
	return self.get_nowait()
```
### def get_nowait
非阻塞从队列中获取数据，如果队列为空则抛出异常。
```python
def get_nowait(self):
	if self.empty():
		raise QueueEmpty
	item = self._get()
	# 尝试唤醒一个向队列中添加数据的程序
	self._wakeup_next(self._putters)
	return item
```
### def task_done
当一个程序从队列中获取一个任务并完成后需要调用队列的`task_done`方法，告诉队列获取的任务已经完成，队列会把未完成的任务计数器减一，当计数器为0时，会解除对调用`join`方法程序的阻塞。
```python
def task_done(self):
	if self._unfinished_tasks <= 0:
		raise ValueError('task_done() called too many times')
	self._unfinished_tasks -= 1
	if self._unfinished_tasks == 0:
		self._finished.set()
```
### def join
调用后`join`方法后，阻塞程序直到队列中所有的任务都完成。
```python
async def join(self):
	if self._unfinished_tasks > 0:
		await self._finished.wait()
```
## class PriorityQueue
```python
class PriorityQueue(Queue):
    def _init(self, maxsize):
        self._queue = []

    def _put(self, item, heappush=heapq.heappush):
        heappush(self._queue, item)

    def _get(self, heappop=heapq.heappop):
        return heappop(self._queue)
```
## class LifoQueue
```python
class LifoQueue(Queue):
    def _init(self, maxsize):
        self._queue = []

    def _put(self, item):
        self._queue.append(item)

    def _get(self):
        return self._queue.pop()
```
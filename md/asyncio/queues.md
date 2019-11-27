# asyncio 之 queues.py
## Queue
### 初始化队列
队列初始化的时候会创建 getters 队列、putters 队列和 queue 数据队列
当 queue 为空时，所有的 get 操作都会放在 getters 队列中等待 put 操作。
当 queue 为满时，所有的 put 操作都会放在 putters 队列中等待 get 操作。
```python
def __init__(self, maxsize=0, *, loop=None):
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
把这两个方法独立出来是为了在子类中重写，以实现不同的队列效果
```python
def _get(self):
	return self._queue.popleft()

def _put(self, item):
	self._queue.append(item)
```
### 删除超时的 get/put future
get 和 put 操作可能会在等待过程中取消，这两个方法就是把取消的 get 或 put 操作从 getters 或 putters 队列中删除。
这两个方法并不会检索队列中的所有对象。而是会从前到后删除直到遇到第一个未取消的对象退出。
```python
def _consume_done_getters(self):
	while self._getters and self._getters[0].done():
		self._getters.popleft()

def _consume_done_putters(self):
	while self._putters and self._putters[0][1].done():
		self._putters.popleft()
```
### 获取队列的状态的方法
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
@coroutine
def put(self, item):
	self._consume_done_getters()
	# 如果 getters 队列不为空，取出一个 get future 让它去队列中获取数据。
	if self._getters:
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')
		getter = self._getters.popleft()
		self._put(item)
		getter.set_result(self._get())

	# 如果队列已满，创建一个 future 添加到 putters 队列中并等待其完成。
	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		waiter = futures.Future(loop=self._loop)
		self._putters.append((item, waiter))
		yield from waiter
	else:
		self._put(item)
```
### def put_nowait
非阻塞向队列中添加数据，如果队列已满，抛出异常。
```python
def put_nowait(self, item):
	self._consume_done_getters()
	if self._getters:
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')

		getter = self._getters.popleft()
		self._put(item)
		getter.set_result(self._get())

	# 如果队列已满 抛出异常
	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		raise QueueFull
	else:
		self._put(item)
```
### def get
从队列获取数据，如果队列为空，阻塞直到队列有数据。
```python
@coroutine
def get(self):
	self._consume_done_putters()
	# 如果 putters 队列不为空，在获取队列数据之后，取出一个 put future 让其完成。
	if self._putters:
		assert self.full(), 'queue not full, why are putters waiting?'
		item, putter = self._putters.popleft()
		self._put(item)
		# 确保 put 任务在 get 任务完成之后完成
		self._loop.call_soon(putter._set_result_unless_cancelled, None)
		return self._get()
	elif self.qsize():
		return self._get()

	# 队列为空，创建一个 future 添加到 getters 队列中并等待其完成。
	else:
		waiter = futures.Future(loop=self._loop)
		self._getters.append(waiter)
		return (yield from waiter)
```
### def get_nowait
非阻塞从队列中获取数据，如果队列为空则抛出异常。
```python
def get_nowait(self):
	self._consume_done_putters()
	if self._putters:
		assert self.full(), 'queue not full, why are putters waiting?'
		item, putter = self._putters.popleft()
		self._put(item)
		putter.set_result(None)
		return self._get()

	elif self.qsize():
		return self._get()

	# 队列为空，抛出异常
	else:
		raise QueueEmpty
```
## PriorityQueue
```python
class PriorityQueue(Queue):
    def _init(self, maxsize):
        self._queue = []

    def _put(self, item, heappush=heapq.heappush):
        heappush(self._queue, item)

    def _get(self, heappop=heapq.heappop):
        return heappop(self._queue)
```
## LifoQueue
```python
class LifoQueue(Queue):
    def _init(self, maxsize):
        self._queue = []

    def _put(self, item):
        self._queue.append(item)

    def _get(self):
        return self._queue.pop()
```
## JoinableQueue
这个队列和线程一样实现了`join`方法，线程的`join`方法阻塞程序到线程结束。而这个队列实现的`join`方法则是阻塞程序直到队列中的任务全部完成。
```python
class JoinableQueue(Queue):
    def __init__(self, maxsize=0, *, loop=None):
        super().__init__(maxsize=maxsize, loop=loop)
		# 队列中未完成任务计数器
        self._unfinished_tasks = 0
        self._finished = locks.Event(loop=self._loop)
        self._finished.set()

	# 向队列中添加一项数据，并把队列中未完成的任务数量增加1
    def _put(self, item):
        super()._put(item)
        self._unfinished_tasks += 1
        self._finished.clear()

	# 当一个程序从队列中获取一个任务并完成后需要调用队列的 task_done() 方法，告诉队列获取的任务已经完成，队列会把未完成的任务计数器减一，当计数器为0时，会解除对调用 join() 方法程序的阻塞。
    def task_done(self):
        if self._unfinished_tasks <= 0:
            raise ValueError('task_done() called too many times')
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._finished.set()

	# 调用后 join() 方法后，阻塞程序直到队列中所有的任务都完成。
    @coroutine
    def join(self):
        if self._unfinished_tasks > 0:
            yield from self._finished.wait()
```
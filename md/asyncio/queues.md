# asyncio 之 queues.py
## 先入先出队列 Queue
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
	# 返回队列中的第一个
	return self._queue.popleft()

def _put(self, item):
	# 添加到队列中的末尾
	self._queue.append(item)
```
### 删除超时的 get/put future
```python
def _consume_done_getters(self):
	# 删除超时或者取消的等待获取队列数据的 future
	while self._getters and self._getters[0].done():
		self._getters.popleft()

def _consume_done_putters(self):
	# 删除超时或者取消的等待添加队列数据的 future
	while self._putters and self._putters[0][1].done():
		self._putters.popleft()
```
### 获取队列的状态的函数
```python
def qsize(self):
	# 返回当前队列的长度
	return len(self._queue)

@property
def maxsize(self):
	# 返回队列的最大长度
	return self._maxsize

def empty(self):
	# 队列为空返回True
	return not self._queue

def full(self):
	# 队列长度达到设置的最大长度返回True，默认不限制队列长度
	if self._maxsize <= 0:
		return False
	else:
		return self.qsize() >= self._maxsize
```
### 向队列添加数据
```python
@coroutine
def put(self, item):
	# 删除已经取消的等待获取数据的 future
	self._consume_done_getters()
	# 如果存在等待获取的 future
	if self._getters:
		# 断言队列一定为空，这样才会有等待获取队列数据的 future
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')
		# 取出一个等待获取数据的 future
		getter = self._getters.popleft()
		# 向队列添加数据
		self._put(item)

		# 从队列获取数据并添加到 future 的结果中
		getter.set_result(self._get())
	# 没有等待获取的 future 并且队列已满
	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		# 创建一个等待向队列中添加数据的 future
		waiter = futures.Future(loop=self._loop)
		# 把要添加的数据和 future 一起添加到 _putters 队列中
		self._putters.append((item, waiter))
		yield from waiter
	# 队列不满，直接向队列添加数据
	else:
		self._put(item)
```
### 非阻塞向队列添加数据
```python
def put_nowait(self, item):
	self._consume_done_getters()
	# 如果 _getters 队列不为空
	if self._getters:
		# 队列必须为空
		assert not self._queue, (
			'queue non-empty, why are getters waiting?')

		getter = self._getters.popleft()
		# 向队列中添加数据
		self._put(item)

		# 从队列中取出数据添加到 future 的结果中
		getter.set_result(self._get())
	# 如果队列已满 抛出异常
	elif self._maxsize > 0 and self._maxsize <= self.qsize():
		raise QueueFull
	# 队列不满 添加数据
	else:
		self._put(item)
```
### 从队列获取数据
```python
@coroutine
def get(self):
	self._consume_done_putters()
	if self._putters:
		# 队列一定是满了
		assert self.full(), 'queue not full, why are putters waiting?'
		# 取出要添加的 数据和future
		item, putter = self._putters.popleft()
		# 向队列中添加数据
		self._put(item)

		# 确保 put 任务在 get 任务完成之后完成
		self._loop.call_soon(putter._set_result_unless_cancelled, None)
		# 返回队列中的数据
		return self._get()
	# 如果队列不为空，直接获取数据
	elif self.qsize():
		return self._get()
	# 队列为空，需要等待
	else:
		waiter = futures.Future(loop=self._loop)
		# 添加一个 get future
		self._getters.append(waiter)
		return (yield from waiter)
```
### 非阻塞从队列中获取数据
```python
def get_nowait(self):
	self._consume_done_putters()
	if self._putters:
		assert self.full(), 'queue not full, why are putters waiting?'
		item, putter = self._putters.popleft()
		self._put(item)

		# 给 put future 设置结果
		putter.set_result(None)
		# 返回获取到的结果
		return self._get()
	# 队列不为空，返回一个队列中的数据
	elif self.qsize():
		return self._get()
	# 队列为空，抛出异常
	else:
		raise QueueEmpty
```
## 优先队列 PriorityQueue
### 初始化
```python
class PriorityQueue(Queue):
    def _init(self, maxsize):
        self._queue = []

    def _put(self, item, heappush=heapq.heappush):
        heappush(self._queue, item)

    def _get(self, heappop=heapq.heappop):
        return heappop(self._queue)
```
## 后入先出队列 LifoQueue
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
```python
class JoinableQueue(Queue):
    def __init__(self, maxsize=0, *, loop=None):
        super().__init__(maxsize=maxsize, loop=loop)
		# 队列中未完成任务计数器
        self._unfinished_tasks = 0
        self._finished = locks.Event(loop=self._loop)
        self._finished.set()

    def _format(self):
        result = Queue._format(self)
        if self._unfinished_tasks:
            result += ' tasks={}'.format(self._unfinished_tasks)
        return result
	# 向队列中添加一项数据，并把队列中未完成的任务数量增加1
    def _put(self, item):
        super()._put(item)
        self._unfinished_tasks += 1
        self._finished.clear()

    def task_done(self):
        # 当一个 get() 完成后调用 task_done() 告诉队列完成了一个任务，队列会把未完成的任务数量减一，当队列中未完成的数量为0时，解除调用 join() 任务的阻塞
        if self._unfinished_tasks <= 0:
            raise ValueError('task_done() called too many times')
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._finished.set()

    @coroutine
    def join(self):
        # 阻塞，直到队列中未完成的任务数量为0
        if self._unfinished_tasks > 0:
            yield from self._finished.wait()
```
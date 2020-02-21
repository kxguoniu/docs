[TOC]
# asyncio 之 selectors.py
## 摘要
`selectors` 在 `select` 模块的基础上进行了一次封装。可以根据系统环境选择当前平台的最优实现。并且在注册的时候添加了一个新的属性`data`，`data`可以是任何类型的数据，但通常我们都会把它作为一个回调函数来使用。
`selectors` 选择的顺序则是 `kqueue` -> `epoll` -> `devpoll` -> `poll` -> `select`。

## 文件内置属性和方法
```python
# 可读事件
EVENT_READ = (1 << 0)
# 可写事件
EVENT_WRITE = (1 << 1)
# 使得文件对象与其文件描述符，关联事件以及附加数据关联起来作为一个整体。
SelectorKey = namedtuple('SelectorKey', ['fileobj', 'fd', 'events', 'data'])
def _fileobj_to_fd(fileobj):
    # 返回文件对象的文件描述符
    if isinstance(fileobj, int):
        fd = fileobj
    else:
        try:
            fd = int(fileobj.fileno())
        except (AttributeError, TypeError, ValueError):
            raise ValueError("Invalid file object: "
                             "{!r}".format(fileobj)) from None
    if fd < 0:
        raise ValueError("Invalid file descriptor: {}".format(fd))
    return fd
```
## class _SelectorMapping
文件对象与选择器的映射关系
`_SelectorMapping`实例化对象作为选择器的一个属性存在，功能是判断已注册文件描述符的数量、获取文件对象注册的信息等。
```python
class _SelectorMapping(Mapping):
    """Mapping of file objects to selector keys."""

    def __init__(self, selector):
		# 选择器对象
        self._selector = selector

    def __len__(self):
		# 返回选择器已经注册的文件描述符数量
        return len(self._selector._fd_to_key)

    def __getitem__(self, fileobj):
		# 获取已经注册文件对象的信息
		# fd； 文件描述符
		# key； SelectorKey 对象
        try:
            fd = self._selector._fileobj_lookup(fileobj)
            return self._selector._fd_to_key[fd]
        except KeyError:
            raise KeyError("{!r} is not registered".format(fileobj)) from None

    def __iter__(self):
        return iter(self._selector._fd_to_key)
```
## class BaseSelector
选择器的抽象类
```python
class BaseSelector(metaclass=ABCMeta):
    @abstractmethod
    def register(self, fileobj, events, data=None):
		# 注册文件对象
		# 文件描述符或者文件对象，注册的事件，附加数据
        raise NotImplementedError

    @abstractmethod
    def unregister(self, fileobj):
		# 取消注册
        raise NotImplementedError

    def modify(self, fileobj, events, data=None):
		# 更改已经注册的文件对象，监视的事件或者附加数据
        self.unregister(fileobj)
        return self.register(fileobj, events, data)

    @abstractmethod
    def select(self, timeout=None):
		# 等待准备好的事件，或者超时返回。
        raise NotImplementedError

    def close(self):
		# 关闭选择器，必须调用用于释放资源
        pass

    def get_key(self, fileobj):
		# 返回与文件对象关联的键
        mapping = self.get_map()
        if mapping is None:
            raise RuntimeError('Selector is closed')
        try:
            return mapping[fileobj]
        except KeyError:
            raise KeyError("{!r} is not registered".format(fileobj)) from None

    @abstractmethod
    def get_map(self):
        # 返回文件对象到选择器的映射
        raise NotImplementedError

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()
```
## class _BaseSelectorImpl
基础的选择器实现
### 初始化
```python
class _BaseSelectorImpl(BaseSelector):
    def __init__(self):
        # this maps file descriptors to keys
		# 文件描述符与 SelectorKey 的映射
        self._fd_to_key = {}
        # 只读的映射
        self._map = _SelectorMapping(self)
```
### def _fileobj_lookup
```python
def _fileobj_lookup(self, fileobj):
	# 尝试返回文件对象的文件描述符
	try:
		return _fileobj_to_fd(fileobj)
	# 如果对象没有文件描述符，尝试搜索
	except ValueError:
		# Do an exhaustive search.
		for key in self._fd_to_key.values():
			if key.fileobj is fileobj:
				return key.fd
		# 抛出异常
		raise
```
### def register
```python
def register(self, fileobj, events, data=None):
	if (not events) or (events & ~(EVENT_READ | EVENT_WRITE)):
		raise ValueError("Invalid events: {!r}".format(events))

	# 创建一个 SelectorKey 对象
	key = SelectorKey(fileobj, self._fileobj_lookup(fileobj), events, data)

	if key.fd in self._fd_to_key:
		raise KeyError("{!r} (FD {}) is already registered"
					   .format(fileobj, key.fd))

	# 文件描述符到key的字典
	self._fd_to_key[key.fd] = key
	return key
```
### def unregister
```python
def unregister(self, fileobj):
	try:
		key = self._fd_to_key.pop(self._fileobj_lookup(fileobj))
	except KeyError:
		raise KeyError("{!r} is not registered".format(fileobj)) from None
	return key
```
### def modify
```python
def modify(self, fileobj, events, data=None):
	# 尝试取出注册的信息
	try:
		key = self._fd_to_key[self._fileobj_lookup(fileobj)]
	except KeyError:
		raise KeyError("{!r} is not registered".format(fileobj)) from None
	# 如果是监视的时间不同
	if events != key.events:
		self.unregister(fileobj)
		key = self.register(fileobj, events, data)
	# 如果是负载的数据不同
	elif data != key.data:
		# 使用快捷方式更新数据
		key = key._replace(data=data)
		self._fd_to_key[key.fd] = key
	return key
```
### def close
```python
def close(self):
	self._fd_to_key.clear()
	self._map = None
```
### def get_map
```python
def get_map(self):
	return self._map
```
### def _key_from_fd
返回与文件描述符关联的数据
```python
def _key_from_fd(self, fd):
	try:
		return self._fd_to_key[fd]
	except KeyError:
		return None
```
## class SelectSelector
### 初始化
```python
class SelectSelector(_BaseSelectorImpl):
    def __init__(self):
        super().__init__()
		# 监视可读事件的文件描述符集合
        self._readers = set()
		# 监视可写事件的文件描述符集合
        self._writers = set()
```
### def register
```python
def register(self, fileobj, events, data=None):
	key = super().register(fileobj, events, data)
	if events & EVENT_READ:
		self._readers.add(key.fd)
	if events & EVENT_WRITE:
		self._writers.add(key.fd)
	return key
```
### def unregister
```python
def unregister(self, fileobj):
	key = super().unregister(fileobj)
	self._readers.discard(key.fd)
	self._writers.discard(key.fd)
	return key
```
### def _select
```python
if sys.platform == 'win32':
	def _select(self, r, w, _, timeout=None):
		r, w, x = select.select(r, w, w, timeout)
		return r, w + x, []
else:
	_select = select.select
```
### def select
```python
    def select(self, timeout=None):
        timeout = None if timeout is None else max(timeout, 0)
        ready = []
		# 返回注册成功的列表
        try:
            r, w, _ = self._select(self._readers, self._writers, [], timeout)
        except InterruptedError:
            return ready
        r = set(r)
        w = set(w)
        for fd in r | w:
            events = 0
            if fd in r:
                events |= EVENT_READ
            if fd in w:
                events |= EVENT_WRITE
			# 取出文件描述符注册时的数据
            key = self._key_from_fd(fd)
			# 添加到准备好的列表中
            if key:
                ready.append((key, events & key.events))
        return ready
```
## class PollSelector
### 初始化
```python
if hasattr(select, 'poll'):

    class PollSelector(_BaseSelectorImpl):
        def __init__(self):
            super().__init__()
            self._poll = select.poll()
```
### def regiser
```python
def register(self, fileobj, events, data=None):
	key = super().register(fileobj, events, data)
	poll_events = 0
	if events & EVENT_READ:
		poll_events |= select.POLLIN
	if events & EVENT_WRITE:
		poll_events |= select.POLLOUT
	# 使用 poll 注册
	self._poll.register(key.fd, poll_events)
	return key
```
### def unregiser
```python
def unregister(self, fileobj):
	key = super().unregister(fileobj)
	# 取消 poll 中的注册
	self._poll.unregister(key.fd)
	return key
```
### def select
```python
def select(self, timeout=None):
	if timeout is None:
		timeout = None
	elif timeout <= 0:
		timeout = 0
	else:
		# poll 的分辨率是毫秒
		timeout = math.ceil(timeout * 1e3)
	ready = []
	# 取出已经准备好的文件描述符
	try:
		fd_event_list = self._poll.poll(timeout)
	except InterruptedError:
		return ready
	for fd, event in fd_event_list:
		events = 0
		if event & ~select.POLLIN:
			events |= EVENT_WRITE
		if event & ~select.POLLOUT:
			events |= EVENT_READ
		# 取出注册时的数据
		key = self._key_from_fd(fd)
		if key:
			ready.append((key, events & key.events))
	return ready
```
## class EpollSelector
### 初始化
```python
if hasattr(select, 'epoll'):

    class EpollSelector(_BaseSelectorImpl):
        def __init__(self):
            super().__init__()
            self._epoll = select.epoll()
```
### def fileno
```python
def fileno(self):
	return self._epoll.fileno()
```
### def regiser
```python
def register(self, fileobj, events, data=None):
	key = super().register(fileobj, events, data)
	epoll_events = 0
	if events & EVENT_READ:
		epoll_events |= select.EPOLLIN
	if events & EVENT_WRITE:
		epoll_events |= select.EPOLLOUT
	# 使用 epoll 注册
	self._epoll.register(key.fd, epoll_events)
	return key
```
### def unregiser
```python
def unregister(self, fileobj):
	key = super().unregister(fileobj)
	try:
		self._epoll.unregister(key.fd)
	except OSError:
		# fd 在注册之后就关闭了
		pass
	return key
```
### def select
```python
def select(self, timeout=None):
	if timeout is None:
		timeout = -1
	elif timeout <= 0:
		timeout = 0
	else:
		# epoll_wait() has a resolution of 1 millisecond, round away
		# from zero to wait *at least* timeout seconds.
		timeout = math.ceil(timeout * 1e3) * 1e-3

	# epoll_wait() expects `maxevents` to be greater than zero;
	# we want to make sure that `select()` can be called when no
	# FD is registered.
	max_ev = max(len(self._fd_to_key), 1)

	ready = []
	try:
		fd_event_list = self._epoll.poll(timeout, max_ev)
	except InterruptedError:
		return ready
	for fd, event in fd_event_list:
		events = 0
		if event & ~select.EPOLLIN:
			events |= EVENT_WRITE
		if event & ~select.EPOLLOUT:
			events |= EVENT_READ

		key = self._key_from_fd(fd)
		if key:
			ready.append((key, events & key.events))
	return ready
```
### def close
```python
def close(self):
	self._epoll.close()
	super().close()
```
## class DevpollSelector
```python
if hasattr(select, 'devpoll'):

    class DevpollSelector(_BaseSelectorImpl):
        """Solaris /dev/poll selector."""

        def __init__(self):
            super().__init__()
            self._devpoll = select.devpoll()

        def fileno(self):
            return self._devpoll.fileno()

        def register(self, fileobj, events, data=None):
            key = super().register(fileobj, events, data)
            poll_events = 0
            if events & EVENT_READ:
                poll_events |= select.POLLIN
            if events & EVENT_WRITE:
                poll_events |= select.POLLOUT
            self._devpoll.register(key.fd, poll_events)
            return key

        def unregister(self, fileobj):
            key = super().unregister(fileobj)
            self._devpoll.unregister(key.fd)
            return key

        def select(self, timeout=None):
            if timeout is None:
                timeout = None
            elif timeout <= 0:
                timeout = 0
            else:
                # devpoll() has a resolution of 1 millisecond, round away from
                # zero to wait *at least* timeout seconds.
                timeout = math.ceil(timeout * 1e3)
            ready = []
            try:
                fd_event_list = self._devpoll.poll(timeout)
            except InterruptedError:
                return ready
            for fd, event in fd_event_list:
                events = 0
                if event & ~select.POLLIN:
                    events |= EVENT_WRITE
                if event & ~select.POLLOUT:
                    events |= EVENT_READ

                key = self._key_from_fd(fd)
                if key:
                    ready.append((key, events & key.events))
            return ready

        def close(self):
            self._devpoll.close()
            super().close()
```

## class KqueueSelector
```python
if hasattr(select, 'kqueue'):

    class KqueueSelector(_BaseSelectorImpl):
        """Kqueue-based selector."""

        def __init__(self):
            super().__init__()
            self._kqueue = select.kqueue()

        def fileno(self):
            return self._kqueue.fileno()

        def register(self, fileobj, events, data=None):
            key = super().register(fileobj, events, data)
            if events & EVENT_READ:
                kev = select.kevent(key.fd, select.KQ_FILTER_READ,
                                    select.KQ_EV_ADD)
                self._kqueue.control([kev], 0, 0)
            if events & EVENT_WRITE:
                kev = select.kevent(key.fd, select.KQ_FILTER_WRITE,
                                    select.KQ_EV_ADD)
                self._kqueue.control([kev], 0, 0)
            return key

        def unregister(self, fileobj):
            key = super().unregister(fileobj)
            if key.events & EVENT_READ:
                kev = select.kevent(key.fd, select.KQ_FILTER_READ,
                                    select.KQ_EV_DELETE)
                try:
                    self._kqueue.control([kev], 0, 0)
                except OSError:
                    # This can happen if the FD was closed since it
                    # was registered.
                    pass
            if key.events & EVENT_WRITE:
                kev = select.kevent(key.fd, select.KQ_FILTER_WRITE,
                                    select.KQ_EV_DELETE)
                try:
                    self._kqueue.control([kev], 0, 0)
                except OSError:
                    # See comment above.
                    pass
            return key

        def select(self, timeout=None):
            timeout = None if timeout is None else max(timeout, 0)
            max_ev = len(self._fd_to_key)
            ready = []
            try:
                kev_list = self._kqueue.control(None, max_ev, timeout)
            except InterruptedError:
                return ready
            for kev in kev_list:
                fd = kev.ident
                flag = kev.filter
                events = 0
                if flag == select.KQ_FILTER_READ:
                    events |= EVENT_READ
                if flag == select.KQ_FILTER_WRITE:
                    events |= EVENT_WRITE

                key = self._key_from_fd(fd)
                if key:
                    ready.append((key, events & key.events))
            return ready

        def close(self):
            self._kqueue.close()
            super().close()
```
## 自动选择平台最优实现
```python
if 'KqueueSelector' in globals():
    DefaultSelector = KqueueSelector
elif 'EpollSelector' in globals():
    DefaultSelector = EpollSelector
elif 'DevpollSelector' in globals():
    DefaultSelector = DevpollSelector
elif 'PollSelector' in globals():
    DefaultSelector = PollSelector
else:
	DefaultSelector = SelectSelector
```
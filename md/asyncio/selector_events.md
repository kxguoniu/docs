[TOC]
# asyncio 之 select_events.py
## 摘要

## class _SelectorTransport
### 初始化
```python
class _SelectorTransport(transports._FlowControlMixin,
                         transports.Transport):
	# 接受数据的缓存大小
    max_size = 256 * 1024  # Buffer size passed to recv().
	# 缓冲区使用的函数
    _buffer_factory = bytearray  # Constructs initial value for self._buffer.

    _sock = None

    def __init__(self, loop, sock, protocol, extra=None, server=None):
        super().__init__(extra, loop)
		# 额外的套接字信息
        self._extra['socket'] = sock
        try:
            self._extra['sockname'] = sock.getsockname()
        except OSError:
            self._extra['sockname'] = None
        if 'peername' not in self._extra:
            try:
                self._extra['peername'] = sock.getpeername()
            except socket.error:
                self._extra['peername'] = None
        self._sock = sock
        self._sock_fd = sock.fileno()

        self._protocol_connected = False
        self.set_protocol(protocol)

        self._server = server
        self._buffer = self._buffer_factory()
        self._conn_lost = 0  # Set when call to connection_lost scheduled.
        self._closing = False  # Set when close() called.
        if self._server is not None:
            self._server._attach()
        loop._transports[self._sock_fd] = self
```
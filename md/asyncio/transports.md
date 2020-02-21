[TOC]
# asyncio 之 transports.py
## 摘要
该文件中定义了基础传输接口，只读传输接口，只写传输接口，读写传输接口，报文传输接口的抽象类。
以及一个写入流控制混合基类的实现。
## class BaseTransport
基础的传输类
```python
class BaseTransport:
    def __init__(self, extra=None):
        if extra is None:
            extra = {}
        self._extra = extra

    def get_extra_info(self, name, default=None):
        return self._extra.get(name, default)

    def is_closing(self):
        raise NotImplementedError

	# 关闭传输，缓存的数据将继续发送。
    def close(self):
        raise NotImplementedError

    def set_protocol(self, protocol):
        raise NotImplementedError

    def get_protocol(self):
        raise NotImplementedError
```
## class ReadTransport
只读传输的接口
```python
class ReadTransport(BaseTransport):
	# 如果传输正在读取数据返回True
    def is_reading(self):
        raise NotImplementedError

	# 暂停接受数据
    def pause_reading(self):
        raise NotImplementedError

	# 恢复接受数据
    def resume_reading(self):
        raise NotImplementedError
```
## class WriteTransport
只写传输的接口
```python
class WriteTransport(BaseTransport):
	# 设置缓冲区的最小值和最大值，它们控制暂停写入和恢复写入两个方法。
    def set_write_buffer_limits(self, high=None, low=None):
        raise NotImplementedError

	# 缓冲区的当前的大小
    def get_write_buffer_size(self):
        raise NotImplementedError

	# 向传输中写入数据
    def write(self, data):
        raise NotImplementedError

	# 向传输中写入一个可迭代类型的数据
    def writelines(self, list_of_data):
        data = b''.join(list_of_data)
        self.write(data)

	# 把缓冲区的数据发送完毕后关闭
    def write_eof(self):
        raise NotImplementedError

	# 如果传输支持 write_eof 方法则返回True
    def can_write_eof(self):
        raise NotImplementedError

	# 强制关闭传输，缓冲区的数据将被丢弃
    def abort(self):
        raise NotImplementedError
```
## class Transport(ReadTransport, WriteTransport)
传输的读写接口
## class DatagramTransport
报文传输的接口
```python
class DatagramTransport(BaseTransport):
    def sendto(self, data, addr=None):
        raise NotImplementedError

    def abort(self):
        raise NotImplementedError
```
## class _FlowControlMixin
写入流控制的混合基类
```python
class _FlowControlMixin(Transport):
    def __init__(self, extra=None, loop=None):
        super().__init__(extra)
        assert loop is not None
        self._loop = loop
		# 协议暂停的状态
        self._protocol_paused = False
        self._set_write_buffer_limits()

	# 检查是否需要暂停协议
    def _maybe_pause_protocol(self):
		# 获取缓冲区当前大小
        size = self.get_write_buffer_size()
        if size <= self._high_water:
            return
		# 缓冲区超过上限，并且协议没有暂停
        if not self._protocol_paused:
			# 设置暂停状态
            self._protocol_paused = True
			# 暂停写入缓存区
            try:
                self._protocol.pause_writing()
            except Exception as exc:
                self._loop.call_exception_handler({
                    'message': 'protocol.pause_writing() failed',
                    'exception': exc,
                    'transport': self,
                    'protocol': self._protocol,
                })

	# 检查是否可以继续协议
    def _maybe_resume_protocol(self):
		# 如果协议处于暂停状态并且缓冲区大小小于下限值
        if (self._protocol_paused and
                self.get_write_buffer_size() <= self._low_water):
			# 取消暂停状态
            self._protocol_paused = False
			# 开始写入缓存区
            try:
                self._protocol.resume_writing()
            except Exception as exc:
                self._loop.call_exception_handler({
                    'message': 'protocol.resume_writing() failed',
                    'exception': exc,
                    'transport': self,
                    'protocol': self._protocol,
                })

	# 获取缓存区的设置，上下限
    def get_write_buffer_limits(self):
        return (self._low_water, self._high_water)

	# 设置缓存区的上下限
    def _set_write_buffer_limits(self, high=None, low=None):
        if high is None:
            if low is None:
                high = 64 * 1024
            else:
                high = 4 * low
        if low is None:
            low = high // 4

        if not high >= low >= 0:
            raise ValueError(
                f'high ({high!r}) must be >= low ({low!r}) must be >= 0')

        self._high_water = high
        self._low_water = low

	# 对外开放的设置缓存区大小方法
    def set_write_buffer_limits(self, high=None, low=None):
        self._set_write_buffer_limits(high=high, low=low)
        self._maybe_pause_protocol()

    def get_write_buffer_size(self):
        raise NotImplementedError
```
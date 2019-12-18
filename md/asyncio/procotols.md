[TOC]
# asyncio 之 protocols.py
## 摘要
协议
## class BaseProtocol
协议接口的公共基类
```python
class BaseProtocol:
    def connection_made(self, transport):
        # 建立连接的时候调用

    def connection_lost(self, exc):
        # 关闭连接的时候调用

    def pause_writing(self):
		# 暂停写入缓存区

    def resume_writing(self):
        # 可以写入缓存区
```
## class Protocol
流传输的接口
```python
class Protocol(BaseProtocol):
    """
      start -> CM [-> DR*] [-> ER?] -> CL -> end
    * CM: connection_made()
    * DR: data_received()
    * ER: eof_received()
    * CL: connection_lost()
    """

    def data_received(self, data):
		# 接受数据的时候调用，参数是一个字节对象

    def eof_received(self):
		# 当另一端调用 write_eof 或者等效函数时调用
        # 当返回一个False，传输将关闭自己，如果返回一个True，传输的关闭将取决于协议
```
## class BufferedProtocol
手动控制缓冲区的流协议接口，3.7添加
事件循环可以使用协议提供的接受缓冲区来避免不必要的数据复制。
```python
class BufferedProtocol(BaseProtocol):
    """
      start -> CM [-> GB [-> BU?]]* [-> ER?] -> CL -> end
    * CM: connection_made()
    * GB: get_buffer()
    * BU: buffer_updated()
    * ER: eof_received()
    * CL: connection_lost()
    """

    def get_buffer(self, sizehint):
		# 分配一个新的接受缓存区

    def buffer_updated(self, nbytes):
		# 使用接受到的数据更新缓冲区，参数是写入缓冲区的总字节数

    def eof_received(self):
		#
```
## class DatagramProtocol
报文协议接口
```python
class DatagramProtocol(BaseProtocol):
    def datagram_received(self, data, addr):
        # 接受数据时调用

    def error_received(self, exc):
		# 当发送或者接收出现异常时调用
```
## class SubprocessProtocol
子进程协议接口
```python
class SubprocessProtocol(BaseProtocol):
    def pipe_data_received(self, fd, data):
		# 当子进程将数据写入管道中调用
        # 文件描述符与字节数据

    def pipe_connection_lost(self, fd, exc):
		# 当与子进程关联的文件描述符关闭时调用

    def process_exited(self):
		# 子进程退出时调用
```
## def _feed_data_to_buffered_proto
使用缓冲区协议发送数据
```python
def _feed_data_to_buffered_proto(proto, data):
    data_len = len(data)
    while data_len:
        buf = proto.get_buffer(data_len)
        buf_len = len(buf)
        if not buf_len:
            raise RuntimeError('get_buffer() returned an empty buffer')

        if buf_len >= data_len:
            buf[:data_len] = data
            proto.buffer_updated(data_len)
            return
        else:
            buf[:buf_len] = data[:buf_len]
            proto.buffer_updated(buf_len)
            data = data[buf_len:]
            data_len = len(data)
```
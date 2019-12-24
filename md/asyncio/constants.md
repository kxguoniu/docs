[TOC]
# asyncio 之 constants.py
## 摘要
## 内容
```python
import enum

# 在连接丢失之后，在多少次写入失败后记录警告信息 write()
LOG_THRESHOLD_FOR_CONNLOST_WRITES = 5

# 重试 accept() 之前等待的秒数
ACCEPT_RETRY_DELAY = 1

# 在调试模式中捕获的堆栈数目，数字越大，操作越慢
# (see extract_stack() in format_helpers.py).
DEBUG_STACK_DEPTH = 10

# 等待ssl握手完成的秒数
# The default timeout matches that of Nginx.
SSL_HANDSHAKE_TIMEOUT = 60.0

# Used in sendfile fallback code.  We use fallback for platforms
# that don't support sendfile, or for TLS connections.
SENDFILE_FALLBACK_READBUFFER_SIZE = 1024 * 256

# 枚举类用于在 base_events 和 sslproto 之间打破循环依赖
class _SendfileMode(enum.Enum):
    UNSUPPORTED = enum.auto()
    TRY_NATIVE = enum.auto()
    FALLBACK = enum.auto()
```
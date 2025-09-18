[TOC]
## 摘要
```python
    logger
        判断级别
        创建事件对象
        判断开启状态
        对象交给Filters筛选
        判断handle level 然后 对象交给Handles处理
        是否需要透传

    Handle
        对象交给Filters筛选
        组装日志信息
        记录
```
## 日志配置
```python
LOGGING = {
    "version": 1,
    "incremental": False,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            ...
        }
    },
    "filters": {
        "log_info": {
            ...
        }
    },
    "handlers": {
        "console": {
            ...
        }
    },
    "loggers": {
        "django": {
            ...
        }
    },
    "root": {
        ...
    }
}
```
## 日志格式(formatter)
### 格式参数
### 配置示例
```python
import logging
class Custom1Formatter(logging.Formatter):
    # 自定义更改
    pass

class Custom2Formatter(logging.BufferingFormatter):
    # 同时执行多个格式化对象
    pass

class Custom():
    # 自定义格式化工厂类
    pass

def Custom2(*args, **kwargs):
    # 自定义工厂函数
    return logging.Formatter(*args, **kwargs)

"formatters": {
    "fmt1": {
        "style": "%",
        "format": "%(asctime)s %(funcName)s %(lineno)d %(levelname)s %(message)s",
        "datefmt": "%Y-%m-%d %H:%M:%S",
        "class": "logging.Formatter"
    },
    "fmt2": {
        "style": "$",
        "format": "$(asctime) $(funcName) $(lineno) $(levelname) $(message)",
        "datefmt": "%Y-%m-%d %H:%M:%S",
        "class": CustomFormatter
    },
    "fmt3": {
        "style": "{",
        "format": "{asctime} {funcName} {lineno} {levelname} {message}",
        "datefmt": "%Y-%m-%d %H:%M:%S",
        "class": Custom2Formatter
    },
    "fmt4": {
        "()": Custom,               # 自定义工厂
        "arg1": "value1",           # 工厂参数
        "arg2": "value2",           # 工厂参数
        ".": {"extra1": "value1"}   # 设置实例化对象的额外属性
    }
}

```
## 日志筛选(filter)
```python
class FLevel(object):
    # 自定义筛选日志
    def __init__(self, level_name=None):
        self.level_name = level_name

    def filter(self, recard):
        return True if recard.levelname == self.level_name else False

"filters": {
    "fts1": {
        "name": "A.B"       # 允许 A.B  A.B.* 通过筛选
    },
    "fts2": {
        "()": FLevel,       # 允许特定级别的log通过
        "level_name": "INFO"
    }
}
```
## 日志处理对象(handler)
### StreamHandler
```python
参数 stream 默认 sys.err
方法 setLevel  setFormatter  setStream
```
### FileHandler
filename；文件路径
mode；打开方式
encoding；编码方式
delay；延迟打开文件
方法 setLevel  setFormatter  setStream 文件对象
### RotatingFileHandler
filename; 日志文件路径
mode; 打开模式
maxBytes; 文件最大大小
backupCount; 备份数量
encoding; 文件编码
delay; 延迟打开
### TimedRotatingFileHandler
filename; 日志文件路径
when; 滚动间隔， S M H D->秒 分 时 天; midnight:午夜; W{0-6}: 周一到周日
interval; 滚动间隔  when × interval
backupCount; 备份数量
encoding; 文件编码
delay; 延迟打开
utc; 使用 utc 时间
atTime; 在什么时间反转，只对午夜和周有效，指定翻转的时间。
### WatchedFileHandler
监控文件的设备或者inode是否发生更改
filename；文件路径
mode；打开方式
encoding；编码方式
delay；延迟打开文件
### 示例
```python
import logging
class Custom(logging.handlers.TimedRotatingFileHandler):
    pass

"handlers": {
    "hd1": {
        "level": "INFO",
        "filters": ["fts2"],
        "formatter": "fmt1",
        "class": "logging.FileHandler",
        "filename":"**/python.log",
        "encoding":"utf-8",
        "delay": True
    },
    "hd2": {
        "level": "Debug",
        "filters": ["fts1", "fts2"],
        "formatter": "fmt1",
        "class": "logging.handlers.RotatingFileHandler",
        "filename": "**/python.log",
        "maxBytes": 1024 * 1024 * 10,
        "backupCount": 3
    },
    "hd3": {
        "level": "INFO",
        "filters": [],
        "formatter": "fmt1",
        "()": Custom,
        ".": {},
        "custom_key1": "value1",
        "custom_key2": "value2"
    }
}
```
## 日志对象(logger)
```python
"loggers": {
    "console": {
        "level": "DEBUG",
        "handlers": ["hd1"],
        "filters": [],
        "propagate": False      # 日志是否向上传递
    },
    "python": {
        "level": "DEBUG",
        "handlers": ["hd1"],
        "filters": [],
        "propagate": False      # 日志是否向上传递
    },
    "python.A": {
        "level": "DEBUG",
        "handlers": ["hd2"],
        "filters": [],
        "propagate": True
    },
    "python.A.B": {
        "level": "DEBUG",
        "handlers": ["hd3"],
        "filters": [],
        "propagate": True
    }
}
```
## 根日志对象(root)
```python
"root": {
    "level": "INFO",
    "handlers": [],
    "filters": []
}
```
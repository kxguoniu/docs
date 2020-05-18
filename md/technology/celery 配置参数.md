[TOC]
## 摘要
`Celery`(芹菜)是一个异步任务队列/基于分布式消息传递的作业队列。它侧重于实时操作，但对调度支持也很好。`Celery`用于生产系统每天处理数百万计的任务。`Celery`是`Python`编写的，但是该协议可以在任何语言实现。它可以与其他语言通过`webhooks`实现。`Celery`本身不支持消息服务，官方建议使用`RabbitMQ`提供消息服务，同时也支持`Redis`、数据库等。
## 公共配置
```python
# 允许的数据类型  内容类型/序列化程序名称  eg: application/json
accept_content = {'json','pickle','yaml'}   # type:set,list,tuple; default:json; version:3.0.18

# 结果后端允许的数据类型, 默认与上面使用相同的序列化器
result_accept_content = None    # type:set,list,tuple; default:None; version:4.3
```

## 时间和日期
```python
# 日期和时间类型将使用utc时区
enable_utc = True   # type:bool; default:True; version:2.5 add, 3.0 enable

# 使用自定义时区
timezone = "UTC"        # type:pytz library; default:UTC; version:2.5
```

## 任务配置
任务的执行参数，序列化，压缩，重试等配置
```python
# 重写配置中任何任务的属性
task_annotations = None     # type:dict; default:None; version:2.5
"""
匹配指定任务
{"tasks.add": {"rate_limit": "10/s"}}

匹配所有任务
{"*": {"rate_limit": "10/s"}}

更改函数
def my_on_failure(self, exc, task_id, args, kwargs, einfo):
    print("Task failed: {0!r}".format(exc))
{"*": {"on_failure": my_on_failure}}

使用对象
class MyAnnotate(object):
    def annotate(self, task):
        if task.name.startswith("tasks."):
            return {"rate_limit": "10/s"}
(MyAnnotate(), {other,})
"""

# 任务数据的默认压缩方法，gzip，bzip2, komubu 注册表中的任何自定义
task_compression = None # type:str; default:None; version:

# 发送任务的消息协议版本 1 or 2
task_protocol = 2   # type:int; default:(4.x and 3.1.24 is 2; other is 1)

# 任务数据的序列化器   json,pickle,yaml,msgpack, kombu 注册的序列化器
task_serializer = "json"    # type:str; default:(4.x is json; other is pickle)

# 连接丢失等连接错误下是否尝试重试
task_publish_retry = True   # type:bool; default:True; version:2.2

# 重试参数设置
task_publish_retry_policy = {   # type:dict; default:; version:2.2
    # 默认重试三次， None 值表示一直重试
    'max_retries': 3,
    # 重试之间等待的秒数
    'interval_start': 0,
    # 每次重试增加的秒数
    'interval_step': 0.2
    # 重试之间等待的最大秒数
    'interval_max': 1,
}
```

## 任务执行配置
任务执行、超时、结果保存设置
```python
# 如果为真，任务将一直在本地阻塞执行
task_always_eager = False

# 如果为真， 任务在本地执行将会抛出异常
task_eager_propagates = False

# 如果为真， 任务结果将包括重新引发的异常堆栈  需要 tblib支持 pip install celery[tblib]
task_remote_tracebacks = False

# 不存储任务返回的结果
task_ignore_result = False

# 如果开启，在结果中存储异常，即使前一项设置为 True
task_store_errors_even_if_ignored = False

# 如果为真，当任务被执行时它会报告，一般不会启用，当存在长时间运行任务并需要报告当前正在运行的任务时非常有用
task_track_started = False

# 超过此值时，执行任务的worker将被杀死替换成新的worker，单位秒。
task_time_limit = None

# 任务执行时间超出将会抛出  SoftTimeLimitExceeded 异常
task_soft_time_limit = None

# 如果为真， ack消息将在任务执行之后确认(默认在执行前确认)，如果任务在执行过程中崩溃，下次重启时会再次被执行
task_acks_late = False

# 启用，即使任务执行失败或者超时也会被确认
task_acks_on_failure_or_timeout = True

# 即使 task_acks_late 被启用，当进程退出或者被杀掉时也会确认消息
# warning: 启用会导致消息循环，确保你知道你在做什么
task_reject_on_worker_lost = None

# 全局缺省限制，只适用于没有设置任务，任务频率限制
task_default_rate_limit = None
```

## 任务结果配置
结果序列化，压缩，结果扩展，过期设置
```python
# 传递给基础传输的附加选项字典 eg： redis {}
result_backend_transport_options = {'visibility_timeout': 18000}

# 结果序列化
result_serializer = "json"      # type:str; default:(4.x is json; other is pickle)

# 任务结果的压缩方法 see task_compression
result_compression = None

# 允许将扩展的任务结果写入后端 名称,参数,worker,重试,队列,交付信息
result_extended = False

# 结果将在多久后删除(None or 0 表示永不过期)，一个内置的定时任务(每天凌晨四点)执行, 目前支持 AMQP、database、cache、redis、Couchbase
result_expires = datetime.timedelta(days=1)     # type:seconds or timedalta; default: one day

# 结果缓存最大值, -1表示禁用, 0 or None 表示没有限制
result_cache_max = -1

# 连接组的超时秒数， 将在 chord 中产生
result_chord_join_timeout = 3.0
```

## 数据库后端配置
数据库连接配置
```python
database_engine_options = {}

# 短会话生存周期，启用会降低性能，对于低流量有用，数据库连接会因为长时间不活动而失效
database_short_lived_sessions = False

# SQLAlchemy 自定义表模式
database_table_schemas = {}

# SQLAlchemy 自定义表名
database_table_names = {}

# 存储任务结果的后端
result_backend = "redis://localhost:6379/14"
```
### RPC
```python
# 默认情况下禁用(临时消息), 如果设置为True，结果消息将保持不变，代理重启后消息不会丢失
result_backend = 'rpc://'
result_persistent = False
```
### Cache
```python
#------------------------Cache -----------------------------------
"""
    # 单个服务器
    result_backend = 'cache+memcached://127.0.0.1:11211/'
    # 多个服务器
    esult_backend = """
        cache+memcached://172.19.26.240:11211;172.19.26.242:11211/
    """.strip()
    # 只在内存中缓存
    result_backend = 'cache'
    cache_backend = 'memory'
"""
# 可以为 pylibmc 设置选项
cache_backend_options = {}
"""
cache_backend_options = {
    'binary': True,
    'behaviors': {'tcp_nodelay': True},
}
"""

# 不在使用此设置，因为可以在结果后端设置中直接指定缓存后端
cache_backend = ""
```
### Redis
```python
"""
result_backend = 'redis://:password@host:port/db'
result_backend = 'redis://localhost/0'

# URL 编码  # required, optional or none; CERT_REQUIRED,CERT_OPTIONAL,CERT_NONE
# urllib.parse.urlencode({"ssl_cert_reqs":"required", ...})
result_backend = 'rediss://:password@host:port/db?\
    ssl_cert_reqs=required\
    &ssl_ca_certs=%2Fvar%2Fssl%2Fmyca.pem\                  # /var/ssl/myca.pem
    &ssl_certfile=%2Fvar%2Fssl%2Fredis-server-cert.pem\     # /var/ssl/redis-server-cert.pem
    &ssl_keyfile=%2Fvar%2Fssl%2Fprivate%2Fworker-key.pem'   # /var/ssl/private/worker-key.pem
result_backend = 'socket:///path/to/redis.sock'
"""
# redis 后端支持redis，此值必须以字典形式设置， see broker_use_ssl
redis_backend_use_ssl = None

# 发送和检索结果 redis连接池中的最大连接数
# warning 并发连接超过最大连接将引发 ConnectionError
redis_max_connections = None

# 连接套接字超时
redis_socket_connect_timeout = None     # type:int/float; version:4.0.1

# redis 读写操作的套接字超时
redis_socket_timeout = 120.0

# redis 读写重试的超时时间， 如果使用的是 unix socket 则不应该设置这个值
redis_retry_on_timeout = False      # version:4.4.1

# keepalive
redis_socket_keepalive = False

#------------------------------------------Cassandra 后端配置-----------------------------------
#------------------------------------------S3 后端配置------------------------------------------
#------------------------------------------Azure Block Blob 后端设置----------------------------
#------------------------------------------Elasticsearch 后端配置-------------------------------
#------------------------------------------Riak 后端配置----------------------------------------
#------------------------------------------AWS DynamoDB 后端配置--------------------------------
#------------------------------------------IronCache 后端配置-----------------------------------
#------------------------------------------Couchbase 后端配置-----------------------------------
#------------------------------------------ArangoDB 后端配置------------------------------------
#------------------------------------------CosmosDB 后端配置------------------------------------
#------------------------------------------CouchDB 后端配置-------------------------------------
#------------------------------------------File-system 后端配置---------------------------------
#------------------------------------------Consul K/V store 后端配置----------------------------
```

## 消息路由
```python
# 大多数用户不需要使用此设置 而是使用自动路由 
# 如果需要配置这应该是一个 list  [kombu.Queue]
# 可以使用 -O 选项覆盖此设置，也可以使用 -X 排除列表中的队列
task_queues = None

# 路由表，按顺序访问， 可以是函数,字符串(函数的路径),字典,元组
# 寻找路由时 task设置的参数具有优先级
# task_routes 中定义的属性高于 task_queues 中定义的属性
task_routes = None
"""
task_routes = {
    'celery.ping': 'default',
    'mytasks.add': 'cpu-bound',
    'feed.tasks.*': 'feeds',                           # <-- glob pattern
    re.compile(r'(image|video)\.tasks\..*'): 'media',  # <-- regex
    'video.encode': {
        'queue': 'video',
        'exchange': 'media',
        'routing_key': 'media.video.encode',
    },
}

def route_task(self, name, args, kwargs, options, task=None, **kw):
    if task == 'celery.ping':
        # 可以返回一个字符串或者字典，字符串表示 task_queues 中的一个名称, 字典是自定义路由
        return {'queue': 'default'}
task_routes = ('myapp.tasks.route_task', {'celery.ping': 'default})
"""

# 为队列设置高可用设置  brokers:RabbitMQ
task_queue_ha_policy = None
"""
所有节点和可用节点
task_queue_ha_policy = 'all'
task_queue_ha_policy = ['rabbit@host1', 'rabbit@host2']
"""

# brokers:RabbitMQ
task_queue_max_priority = None
task_default_priority = None
# 子任务可以继承父任务的优先级
task_inherit_parent_priority = False

# 每个worker都有一个专用队列
# 可以使用任务制定处理的主机
worker_direct = False
""" task_routes = {'tasks.add': {'exchange': 'C.dq', 'routing_key': 'w1@example.com'}} """

# 自动创建在 task_queues 队列中未定义的指定队列
task_create_missing_queues = True

# 默认队列， 所有未指定路由的任务都将发送到默认队列
task_default_queue = "celery"

# 为所有未指定的密钥交换指定默认队列
task_default_exchange = "task_default_queue"

# 在 task_queues 中没有为键指定交换类型时设置的默认交换类型
task_default_exchange_type = "direct"

#  在 task_queues 中没有为密钥指定时设置默认值
task_default_routing_key = "task_default_queue"

# transient  or persistent  写入硬盘
task_default_delivery_mode = "persistent"
```

## 后端配置
```python
# 默认的代理 url
broker_url = "amqp://"
"""
broker_url = "redis://localhost:5379/15"
broker_url = 'proj.transports.MyTransport://localhost'
broker_url = 'transport://userid:password@hostname:port//;transport://userid:password@hostname:port//'
broker_url = [
    'transport://userid:password@localhost:port//',
    'transport://userid:password@hostname:port//'
]
"""

# 可以分开设置
broker_read_url = "broker_url"
broker_write_url = "broker_url"

# 连接故障的默认转移策略
broker_failover_strategy = "round-robin"
"""
def random_failover_strategy(servers):
    it = list(servers)  # don't modify callers list
    shuffle = random.shuffle
    for _ in repeat(None):
        shuffle(it)
        yield it[0]

broker_failover_strategy = random_failover_strategy
"""

# 心跳  pyamqp 支持
broker_heartbeat = 120.0

# 速率 pyamqp 支持 实际心跳检测时间除以这个值
broker_heartbeat_checkrate = 2.0

# 使用ssl  pyamqp,redis 支持
# 使用 broker_use_ssl=True 时需要小心，默认配置可能不会验证服务器证书
broker_use_ssl = False
"""
# pyamqp
import ssl
broker_use_ssl = {
  'keyfile': '/var/ssl/private/worker-key.pem',
  'certfile': '/var/ssl/amqp-server-cert.pem',
  'ca_certs': '/var/ssl/myca.pem',
  'cert_reqs': ssl.CERT_REQUIRED
}
# redis
broker_use_ssl = {
  'ssl_keyfile': '/var/ssl/private/worker-key.pem',
  'ssl_certfile': '/var/ssl/amqp-server-cert.pem',
  'ssl_ca_certs': '/var/ssl/myca.pem',
  'ssl_cert_reqs': ssl.CERT_REQUIRED
}
"""

# 连接池中可以打开的最大连接数, 如果设置None or 0, 连接池将被禁用, 每次请求都会建立和断开连接
broker_pool_limit = 10      # version:2.3

# 连接超时的时间
broker_connection_timeout = 4.0

# 开启自动重连
broker_connection_retry = True

# 最大重试连接次数，None or 0 表示一直尝试
broker_connection_max_retries = 100

# 设置自定义登录方法
broker_login_method = "AMQPLAIN"

# 传递给基础传输的附加选项字典
broker_transport_options = {}   # version:2.2
```

## worker 配置
```python
# 工作程序启动时需要导入的模块列表
# 导入任务控制模块，信号处理程序和远程控制命令等
imports = []

# 功能类似 imports
include = []

# 执行任务的并发 进程/线程/绿线程 的数量
worker_concurrency = "Number of CPU cores"

# 预取 4*worker_concurrency 条消息, 如果需要关闭预取，请设置为1，None or 0 将会经可能多的预取
worker_prefetch_multiplier = 4

# 
worker_lost_wait = 10.0

# 每一个进程指定最大的任务执行数，默认没有限制
worker_max_tasks_per_child = None

# woker在被替换之前指定的最大常驻内存，默认没有限制，单位 kilobytes
worker_max_memory_per_child = None

# 如果为真，禁用所有速率限制，即使任务设置了速率限制
worker_disable_rate_limits = False

# 用于存储持久工作状态的文件
worker_state_db = None  # /tmp/celery.db

# ETA 调度程序可以休眠的最大时间  eg: 0.1
worker_timer_precision = 1.0

# 指定是否启用远程控制
worker_enable_remote_control = True

# 等待新工作进程启动的超时
worker_proc_alive_timeout = 4.0
```

## events 配置
```python
# 发送与任务相关的事件，便于 flower 等程序的监控
worker_send_task_events = False

# 如果启用, 为每一个任务发送 task-sent 便于 worker 追踪
task_send_sent_event = False    # version: 2.2

# 消息过期时间 amqp 传输支持
event_queue_ttl = 5.0

# 过期时间 amqp 传输支持
event_queue_expires = 60.0

# 事件接受队列名称的前缀
event_queue_prefix = "celeryev"

# 事件交换的名称
# 实验阶段，谨慎使用
event_exchange = "celeryev"

# 时间消息序列化器
event_serializer = "json"
```

## 远程控制命令配置
```python 
# 远程控制命令过期时间，应答也是同样的
control_queue_ttl = 300.0

# 从代理中删除未使用的命令频率
control_queue_expires = 10.0

# 控制命令交换的名称
# 试验阶段，谨慎使用
control_exchange = "celery"
```

## 日志配置
```python
# 默认情况下将删除根纪录器上所有的处理程序， 设置 worker_hijack_root_logger = False. 禁用
# 定制日志记录见文档
worker_hijack_root_logger = True    # version:2.2

# 控制台显示颜色
worker_log_color = True

# 日志格式
worker_log_format = "[%(asctime)s: %(levelname)s/%(processName)s] %(message)s"

# 任务日志格式
worker_task_log_format = "[%(asctime)s: %(levelname)s/%(processName)s] [%(task_name)s(%(task_id)s)] %(message)s"

# err and out 都将重定向到当前记录器
worker_redirect_stdouts = True

# 日志记录级别
worker_redirect_stdouts_level = "WARNING"
```

## 安全配置
```python
# 消息签名的私有秘钥文件  相对或者绝对路径
security_key = None     #version:2.5

# 消息签名的证书文件  相对或者绝对路径
security_certificate = None     # version:2.5

# 证书目录   eg  /etc/certs/*.pem
security_cert_store = None  # version:2.5

# 在使用消息签名时对消息进行摘要
security_digest = sha256    # version:4.3
```

## 自定义组件
```python
# woker 使用的池 类名称  celery.concurrency.prefork:TaskPool
worker_pool = "prefork"

# 如果启用可以使用池重启远程控制命令重新启动工作池
worker_pool_restarts = False

# 类名称
worker_autoscaler = "celery.worker.autoscale:Autoscaler"    # version:2.2

# 消费类名称
worker_consumer = "celery.worker.consumer:Consumer"

# ETA 调度类名称
worker_timer = "kombu.asynchronous.hub.timer:Timer"
```

## 定时任务配置
```python
# 定期任务计划
beat_schedule = {}

# 默认的调度类  eg: django_celery_beat.schedulers:DatabaseScheduler
beat_scheduler = "celery.beat:PersistentScheduler"

# PersistentScheduler 存储周期性任务最后一次运行时间的文件
beat_schedule_filename = "celerybeat-schedule"

# 在数据库同步之前可以调用的周期性任务数量， 0 标识基于时间的同步，默认三分钟。1表示每次调用都会同步
beat_sync_every = 0

# 在查看时间表之前可以睡眠的最大秒数，值是特定于调度器的，默认调度器是300s，django调度器是 5s, jython 会覆盖值为1
beat_max_loop_interval = 0
```
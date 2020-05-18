[TOC]
## 摘要
`redis`是一个高性能的`key-value`数据库，支持数据的持久化，可以将内存中的数据保存在磁盘中，支持数据备份，主从模式。支持`list`、`set`、`zset`、`hash`等结构存储。
## 其他
| 空间 | 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:---:|:----:|:---:|:---:|:---:|
|INCLUDES|include|/path/to/other.conf|导入其他文件的配置|1|1|1|
|MODULES|loadmodule|/path/to/other.so|在启动时加载模块|`0`|1|1|
|SECURITY|requirepass|foobared|设置外部访问的密码，需要一个强度非常高的密码以防止暴力破解|1|1|1|
|SECURITY|rename-command|CONFIG b840fc20|命令重命名，外部访问使用重命名之后的命令|1|1|1|
|CLIENTS|maxclients|10000|最大的客户端连接数，跟系统的文件描述符数量有关|1|1|1|
|LUA SCRIPTING|lua-time-limit|5000|lua脚本最长的执行时间，超过限制redis将使用异常回复查询|1|1|1|
|SLOW LOG|slowlog-log-slower-than|10000|慢日志记录，负数禁用，0记录所有。1000000 = 1s|1|1|1|
|SLOW LOG|slowlog-max-len|128|慢日志队列长度，没有限制但会占用内存|1|1|1|
|LATENCY MONITOR|latency-monitor-threshold|0|延迟监控，0表示禁用，<milliseconds> 表示延迟多长时间将被记录|1|1|1|
|EVENT NOTIFICATION|notify-keyspace-events|""|redis键被操作可以通知订阅的客户端|1|1|1|

## NETWORK
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|bind|127.0.0.1|监听地址|1|1|1|
|protected-mode|yes|保护模式，no:外网可以直接访问, yes:需要配置IP或设置密码|`0`|1|1|
|port|6379|监听端口|1|1|1|
|tcp-backlog|511|tcp端口最大监听队列长度，min(backlog,/proc/sys/net/somaxconn)|1|1|1|
|unixsocket|/var/run/redis/redis.sock|unix套接字路径|1|1|1|
|unixsocketperm|700|unix套接字权限|1|1|1|
|timeout|0|客户端空闲超过阈值时关闭连接|1|1|1|
|tcp-keepalive|300|定时发送ACKs确认连接正常|1|1|1|

## GENERAL
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|daemonize|no|redis守护进程启动|1|1|1|
|supervised|no|可以通过upstart和systemd管理reis守护进程|`0`|1|1|
|pidfile|/var/run/redis_6379.pid|redis守护进程的pid文件|1|1|1|
|loglevel|notice|日志级别|1|1|1|
|logfile|/var/log/redis/redis.log|日志文件路径|1|1|1|
|syslog-enabled|no|开启系统日志|1|1|1|
|syslog-ident|redis|系统日志的标识|1|1|1|
|syslog-facility|local0|系统日志指定设备|1|1|1|
|databases|16|redis数据库数量|1|1|1|
|always-show-logo|yes|启动日志中显示logo|`0`|1|1|

## SNAPSHOTTING
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|save|60 10000|redis持久化同步条件|1|1|1|
|show-writes-on-bgsave-error|yes|快照保存失败将禁止写操作直到保存进程再次工作<br/>如果你已经设置了对redis服务和持久化的<br/>监控你可能需要关闭此功能|1|1|1|
|rdbcompression|yes|使用LZF压缩字符串对象|1|1|1|
|rdbchecksum|yes|RDB版本5之后在文件末尾增加了一个CRC64校验和<br/>确保数据难以破坏，但是会增加10%的性能损耗|1|1|1|
|dbfilename|dump.rdb|数据转储文件名称|1|1|1|
|dir|/var/lib/redis|数据文件保存的文件夹|1|1|1|

## REPLICATION
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|slaveof|masterip masterport|作为另一台服务器的从服务器|1|1|`0`|
|replicaof|masterip masterport|成为另一台服务器的副本|`0`|`0`|1|
|masterauth|master-password|redis连接密码|1|1|1|
|slave-serve-stale-data|yes|失去主服务器的连接或者备份正在进行中<br/>副本将以旧的数据回复客户端的请求|1|1|1|
|slave-read-only|yes|备份服务器只读|1|1|1|
|repl-diskless-sync|no|数据同步方式，默认先把数据保存到磁盘然后开始增量复制<br/>另一种不使用磁盘直接把数据发送给备份服务器|1|1|1|
|repl-diskless-sync-delay|5|无磁盘同步策略等待备份服务器连接到达的时间|1|1|1|
|repl-ping-slave-period|10|备份服务器定时发送ping|1|1|1|
|repl-timeout|60|发送超时、响应超时、IO超时|1|1|1|
|repl-disable-tcp-nodelay|no|no:大带宽占用减少延迟；yes:小带宽占用增加延迟|1|1|1|
|repl-backlog-size|1mb|缓冲区，当备份服务器断开重连后不需要完全同步，只需要部分同步<br/>缓冲区越大服务器可以断开的时间更长|1|1|1|
|repl-backlog-ttl|3600|所有备份服务器断开超过指定时间释放缓冲区|1|1|1|
|slave-priority|100|备份服务器优先级，越小表示权重越高|1|1|1|
|min-slaves-to-write|3|备份节点小于此值 延迟小于等于下值,主节点拒绝写操作|1|1|1|
|min-slaves-max-lag|10|备份节点小于上值 延迟小于等于此值,主节点拒绝写操作|1|1|1|
|slave-announce-ip|5.5.5.5|向主服务器报告自己的ip，防止端口转发和NAT的影响|`0`|1|`0`|
|slave-announce-port|1234|向主服务器报告自己的端口，防止端口转发和NAT的影响|`0`|1|`0`|
|replica-announce-ip|5.5.5.5|向主服务器报告自己的ip，防止端口转发和NAT的影响|`0`|`0`|1|
|replica-announce-port|1234|向主服务器报告自己的端口，防止端口转发和NAT的影响|`0`|`0`|1|

## MEMORY MANAGEMENT
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|maxmemory|bytes|redis最大内存占用|1|1|1|
|maxmemory-policy|noeviction|redis内存清理策略，默认不清理|1|1|1|
|maxmemory-samples|5|LRU算法，3最快但不精确 5居中 10最精确但占用更多CPU|1|1|1|
|replica-ignore-maxmemory|yes|redis5开始副本将忽略内存占用设置<br/>数据只会被主服务器传递删除指令删除|`0`|`0`|1|

## LAZY FREEING
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|lazyfree-lazy-eviction|no|非阻塞方式释放回收命令的内存<br/>如果一个键关联了大量数据释放内存会阻塞|`0`|1|1|
|lazyfree-lazy-expire|no|非阻塞方式释放过期命令的内存|`0`|1|1|
|lazyfree-lazy-server-del|no|非阻塞方式释放删除命令的内存|`0`|1|1|
|slave-lazy-flush|no|非阻塞方式释放备份服务器内存|`0`|1|`0`|
|replica-lazy-flush|no|非阻塞方式释放备份服务器内存|`0`|`0`|1|

## APPEND ONLY MODE
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|appendonly|no|是否开启AOF持久化|1|1|1|
|appendfilename|"appendonly.aof"|AOF文件名称|1|1|1|
|appendfsync|everysec|no:禁止手动调用fsync同步(快)，everysec:每秒调用一次(折中)<br/>always:每次更新都要调用(慢，安全)|1|1|1|
|no-appendfsync-on-rewrite|no|调用fsync会导致阻塞，禁止BGSAVE和BGREWRITEAOF<br/>在主进程中调用fsync，最坏的情况会损失30s日志|1|1|1|
|auto-aof-rewrite-percentage|100|aof文件重写百分比|1|1|1|
|auto-aof-rewrite-min-size|64mb|aof文件重写最小大小|1|1|1|
|aof-load-truncated|yes|aof文件可能在末尾被截断，yes表示加载尽可能多的数据<br/>no表示redis不会启动抛出异常|1|1|1|
|aof-use-rdb-preamble|yes|表示重写aof文件时使用 RDB + AOF，no禁用|`0`|1|1|

## REDIS CLUSTER
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|cluster-enabled|yes|节点集群支持|1|1|1|
|cluster-config-file|nodes-6379.conf|集群节点的配置文件|1|1|1|
|cluster-node-timeout|15000|集群节点超时(不可达)|1|1|1|
|cluster-slave-validity-factor|10|(node-timeout * slave-validity-factor) + repl-ping-slave-period<br/>超过这个时间备份服务器不会启用故障切换<br/>值太大会导致节点数据老旧，太小不利于集群选举|1|1|1|
|cluster-migration-barrier|1|集群从服务器会迁移到没有备份服务器的主服务器上以保证集群稳定性<br/>需要确保当前主服务器至少有几个备份服务器存在才会迁移|1|1|1|
|cluster-require-full-coverage|yes|当一个集群节点故障时集群将会不可用直到故障节点恢复<br/>如果你希望集群的子集继续服务可以设置no|1|1|1|
|cluster-replica-no-failover|no|设置为yes表示当前备份服务器不会执行故障转移|`0`|1|1|

## CLUSTER DOCKER/NAT SUPPORT
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|cluster-announce-ip|10.1.1.5|设置集群节点的ip, 类比设置从服务器的ip|`0`|1|1|
|cluster-announce-port|6379|设置集群节点的端口|`0`|1|1|
|cluster-announce-bus-port|6380|设置集群节点的端口|`0`|1|1|

## ADVANCED CONFIG
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|hash-max-ziplist-entries|512|适用ziplist编码的hash元素数量|1|1|1|
|hash-max-ziplist-value|64|适用ziplist编码的hash元素长度|1|1|1|
|list-max-ziplist-entries|512|适用ziplist编码的列表元素数量|1|`0`|`0`|
|list-max-ziplist-value|64|适用ziplist编码的列表元素长度|1|`0`|`0`|
|zset-max-ziplist-entries|128|适用ziplist编码的有序集合元素数量|1|1|1|
|zset-max-ziplist-entries|64|适用ziplist编码的有序集合元素长度|1|1|1|
|list-max-ziplist-size|-2|-1:4kb,-2:8kb,-3:16kb,-4:32kb,-5:64kb|`0`|1|1|
|list-compress-depth|0|0:禁用压缩,1:忽略头尾压缩中间，2：忽略头2尾2压缩中间<br/>3：忽略头3尾3压缩中间...|`0`|1|1|
|set-max-intset-entries|512|集合只由64位有符号的10进制字符，适用|1|1|1|
|hll-sparse-max-bytes|3000|稀疏编码的长度限制|1|1|1|
|activerehashing|yes|占用1/100的cpu时间来重建哈希表|1|1|1|
|stream-node-max-bytes|4096||`0`|`0`|1|
|stream-node-max-entries|100||`0`|`0`|1|
|client-output-buffer-limit|normal 0 0 0|超过硬限制或者超过软限制达到指定时间断开连接<br/>使用于客户端请求速度大于读取速度|1|1|1|
|client-output-buffer-limit|slave 256mb 64mb 60|备份节点限制|1|1|1|
|client-output-buffer-limit|pubsub 32mb 8mb 60|订阅客户端限制|1|1|1|
|client-query-buffer-limit|1gb|客户端查询缓冲区|`0`|1|1|
|proto-max-bulk-len|512mb|批量请求被限制在512mb|`0`|1|1|
|hz|10|1-500之间，大部分选择10，低延迟环境中选择100，超过100不提倡|1|1|1|
|dynamic-hz|yes|动态配置hz，以提供更好的响应|`0`|`0`|1|
|aof-rewrite-incremental-fsync|yes|aof重写32mb手动调用一次fsync|1|1|1|
|rdb-save-incremental-fsync|yes|RDB 保存32mb手动调用一次fsync|`0`|`0`|1|
|lfu-log-factor|10||`0`|1|1|
|lfu-decay-time|1||`0`|1|1|

## ACTIVE DEFRAGMENTATION
| 参数 | 示例 | 解释 | 3.0 | 4.0 | 5.0 |
|:---:|:---:|:----:|:---:|:---:|:---:|
|activedefrag|yes|开启碎片整理，实现性功能|`0`|1|1|
|active-defrag-ignore-bytes|100mb|启动碎片整理最小的浪费数量|`0`|1|1|
|active-defrag-threshold-lower|10|启动碎片整理的最小碎片百分比|`0`|1|1|
|active-defrag-threshold-upper|100|最大百分比|`0`|1|1|
|active-defrag-cycle-min|25|最小占用CPU百分比|`0`|1|1|
|active-defrag-cycle-max|75|最大占用CPU百分比|`0`|1|1|
|active-defrag-max-scan-fields|1000|扫描的最大字段数量|`0`|`0`|1|

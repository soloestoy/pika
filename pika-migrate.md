## Pika3.2到Redis迁移工具

### 适用版本:
Pika 3.2.0及以上,  单机模式

### 功能
将Pika中的数据在线迁移到Pika、Redis(支持全量、增量同步)

### 开发背景:
之前Pika项目官方提供的pika\_to\_redis工具仅支持离线将Pika的DB中的数据迁移到Pika、Redis, 且无法增量同步, 该工具实际上就是一个特殊的Pika, 只不过成为从库之后, 内部会将从主库获取到的数据转发给Redis，同时并支持增量同步,  实现热迁功能.

### 热迁原理
1. pika-port通过dbsync请求获取主库当前全量db数据, 以及当前db数据所对应的binlog点位
2. 获取到主库当前各个db全量数据之后, 扫描db, 并将db中的数据转发给Redis的对应db（通过select命令选择正确的db）
3. 通过之前获取的binlog的点位向主库进行增量同步, 在增量同步的过程中, 将从主库获取到的binlog重组成Redis命令, 转发给Redis（每个db有单独的binlog，通过识别table_name转换成Redis的select命令发送给正确db）

### 新增配置项
```cpp
###################
## Migrate Settings
###################

target-redis-host : 127.0.0.1
target-redis-port : 6379
target-redis-pwd  : abc

sync-batch-num    : 100
redis-sender-num  : 10
```

### 步骤
1. 考虑到在pika-port在将全量数据写入到Redis这段时间可能耗时很长, 导致主库原先binlog点位已经被清理, 我们首先在主库上执行`config set expire-logs-nums 10000`, 让主库保留10000个Binlog文件(Binlog文件占用磁盘空间, 可以根据实际情况确定保留binlog的数量), 确保后续该工具请求增量同步的时候, 对应的Binlog文件还存在.
2. 修改该工具配置文件的`target-redis-host, target-redis-port, target-redis-pwd, sync-batch-num, redis-sender-num`配置项(`sync-batch-num`是该工具接收到主库的全量数据之后, 为了提升转发效率, 将`sync-batch-num`个数据一起打包发送给Redis, 此外该工具内部可以指定`redis-sender-num`个线程用于转发命令（每个db分别会有`redis-sender-num`个线程负责处理）, 命令通过Key的哈希值被分配到不同的线程中, 所以无需担心多线程发送导致的数据错乱的问题)
3. 使用`pika -c pika.conf`命令启动该工具, 查看日志是否有报错信息
4. 向该工具执行`slaveof ip port force`向主库请求同步, 观察是否有报错信息
5. 在确认主从关系建立成功之后(此时pika-port同时也在向目标Redis转发数据了)通过向主库执行`info Replication`查看主从同步延迟(可在主库写入一个特殊的Key, 然后看在Redis测是否可以立马获取到, 来判断是否数据已经基本同步完毕)

### 注意事项
1. Pika支持不同数据结构采用同名Key, 但是Redis不支持, 所以在有同Key数据的场景下, 以第一个迁移到Redis数据结构为准, 其他同Key数据结构会丢失
2. 该工具只支持热迁移单机模式下, 如果是集群模式, 工具会报错并且退出.
3. 为了避免由于主库Binlog被清理导致该工具触发多次全量同步向Redis写入脏数据, 工具自身做了保护, 单个db在第二次触发全量同步时会报错退出.



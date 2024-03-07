# 简介

为了方便查询使用，对Redis6.0的系统状态信息(info)，和Redis的所有配置(Standalone、Sentinel、Cluster三种模式)进行全面总结

# Info

info命令的使用方法有以下三种：

- info：部分Redis系统状态统计信息。

- info all：全部Redis系统状态统计信息。

- info section：某一块的系统状态统计信息，其中section可以忽略大小写，section可以有如下值：

  * all ：全部Redis系统状态统计信息。

  * server：获取 server 信息

  * clients：获取 clients 信息，如客户端连接数等

  * memory：获取 server 的内存信息，包括当前内存消耗、内存使用峰值

  * persistence：获取 server 的持久化配置信息

  * stats：获取 server 的一些基本统计信息，如处理过的连接数量等

  * replication：获取 server 的主从配置信息

  * cpu：获取 server 的 CPU 使用信息

  * keyspace：获取 server 中各个 DB 的 key 的数量

  * cluster：获取集群节点信息，仅在开启集群后可见

  * commandstas：获取每种命令的统计信息

下面是信息详解：

~~~bash
127.0.0.1:6379> info
# Server
redis_version:6.0.16                      #Redis服务版本
redis_git_sha1:00000000                   #Git SHA
redis_git_dirty:0                         #Git dirty flag
redis_build_id:a016d5a8fa742e25           #Redis build id
redis_mode:standalone                     #运行模式，分为: Cluster、Sentinel、Standalone
os:Linux 3.10.0-1062.el7.x86_64 x86_64    #Redis所在机器的操作系统
arch_bits:64                              #架构(32或64位)
multiplexing_api:epoll                    #Redis所使用的事件处理机制
atomicvar_api:atomic-builtin
gcc_version:9.3.1                         #编译Redis时所使用的GCC版本
process_id:73309                          #Redis服务进程的PID
run_id:25a6bc791e9c33f9a4d526a9e08260a7661317c0    #Redis服务的标识符
tcp_port:6380                             #监听的端口
uptime_in_seconds:934                     #自Redis服务启动以来，运行的秒数
uptime_in_days:0                          #自Redis服务启动以来，运行的天数
hz:10                                     #serverCron每秒运行次数
configured_hz:10
lru_clock:16563116                        #以分钟为单位进行自增的时钟，用于LRU管理
executable:/apps/redis/bin/redis-server
config_file:/apps/redis/6379.conf         #Redis的配置文件
io_threads_active:0
  
# Clients
connected_clients:1                #当前客户端连接数
client_recent_max_input_buffer:8   #当前所有输入缓冲区中占用的最大容量。
client_recent_max_output_buffer:0  #当前所有输出缓冲区中占用的最大容量。
blocked_clients:0                  #正在等待阻塞命令(例如BLPOP等)的客户端数量
tracking_clients:0
clients_in_timeout_table:0
  
# Memory
used_memory:870272                 #Redis分配器分配的内存总量，也就是内部存储的所有数据内存占用量
used_memory_human:849.88K            #以可读的格式返回used_memory
used_memory_rss:8101888            #从操作系统的角度显示Redis进程占用的物理内存总量
used_memory_rss_human:7.73M          #以可读的格式返回used_memory_rss
used_memory_peak:925456            #内存使用的最大值，表示used_memory的峰值
used_memory_peak_human:903.77K       #以可读的格式返回used_memory_peak
used_memory_peak_perc:94.04%       #使用内存达到峰值内存的百分比，即(used_memory / used_memory_peak) *100%
used_memory_overhead:825224        #Redis为了维护数据集的内部机制所需的内存大小，包括所有客户端输出缓冲区、查询缓冲区、AOF重写缓冲区和主从复制的backlog。
used_memory_startup:803400         #Redis服务器启动时消耗的内存
used_memory_dataset:45048          #数据占用的内存大小，即used_memory - sed_memory_overhead
used_memory_dataset_perc:67.36%    #数据占用的内存大小的百分比，(used_memory_dataset / (used_memory - used_memory_startup)) * 100%
allocator_allocated:1146216
allocator_active:1503232
allocator_resident:3739648
total_system_memory:1019645952     #系统内存
total_system_memory_human:972.41M    #以可读的格式返回total_system_memory
used_memory_lua:37888              #Lua引警所消耗的内存大小
used_memory_lua_human:37.00K         #以可读的格式返回used_memory_lua
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0                        #Redis实例的最大内存配置
maxmemory_human:0B                   #以可读的格式返回
maxmemory_policy:noeviction        #当达到maxmemory时的淘汰策略
allocator_frag_ratio:1.31
allocator_frag_bytes:357016
allocator_rss_ratio:2.49
allocator_rss_bytes:2236416
rss_overhead_ratio:2.17
rss_overhead_bytes:4362240
mem_fragmentation_ratio:9.79       #used_memory_rss/used_memory比值，表示内存碎片率
mem_fragmentation_bytes:7274128
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:20496
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0       #Redis所使用的内存分配器。默认为jemalloc
active_defrag_running:0            #是否有活动的defrag任务正在运行，1表示有活动的defrag任务正在运行（defrag:表示内存碎片整理）
lazyfree_pending_objects:0         #0表示不存在延迟释放的挂起对象
  
# Persistence
loading:0                          #服务器是否正在载入持久化文件。是1，否0
rdb_changes_since_last_save:0      #自上次RDB后，Redis数据改动条数，即有多少个写入命令没有持久化
rdb_bgsave_in_progress:0           #bgsave操作是否正在进行中。是1，否0
rdb_last_save_time:1660730111      #上次成功生成RDB的时间戳（也可以使用lastsave命令获取）
rdb_last_bgsave_status:ok          #上次rdb持久化是否成功
rdb_last_bgsave_time_sec:0         #上次成功生成rdb文件耗时秒数
rdb_current_bgsave_time_sec:-1     #如果服务器正在创建rdb文件，那么这个记录的就是当前的创建操作已经耗费的秒数
rdb_last_cow_size:462848           #RDB过程中父进程与子进程相比执行了多少修改(包括读缓冲区，写缓冲区，数据修改等)。
aof_enabled:0                      #是否开启了AOF功能。是1，否0
aof_rewrite_in_progress:0          #标识AOF重写（bgrewriteaof）操作是否在进行中。是1，否0
aof_rewrite_scheduled:0            #AOF重写任务计划，当客户端发送bgrewriteaof指令，如果当前rewrite子进程正在执行，那么将客户端请求的bgrewriteaof变为计划任务，待aof子进程结束后执行bgrewriteaof
aof_last_rewrite_time_sec:-1       #上次AOF重写耗费的时长(单位是秒)
aof_current_rewrite_time_sec:-1    #如果AOF重写操作正在进行，则记录当前AOF重写所使用的时间(单位是秒)
aof_last_bgrewrite_status:ok       #上次AOF重写操作的状态
aof_last_write_status:ok           #上次AOF写磁盘的结果
aof_last_cow_size:0                #AOF过程中父进程与子进程相比执行了多少修改(包括读缓冲区，写缓冲区，数据修改等)。
module_fork_in_progress:0
module_fork_last_cow_size:0
  
# Stats
total_connections_received:1       #连接过的客户端总数
total_commands_processed:17        #执行过的命令总数
instantaneous_ops_per_sec:0        #每秒处理命令条数
total_net_input_bytes:1159         #输入总网络流量(以字节为单位)
total_net_output_bytes:23597       #输出总网络流量(以字节为单位)
instantaneous_input_kbps:0.00      #每秒输人字节数
instantaneous_output_kbps:0.00     #每秒输出字节数
rejected_connections:0             #拒绝的连接个数
sync_full:0                        #主从完全同步成功次数
sync_partial_ok:0                  #主从部分同步成功次数
sync_partial_err:0                 #主从部分同步失败次数
expired_keys:0                     #过期的key数量
expired_stale_perc:0.00
expired_time_cap_reached_count:0
expire_cycle_cpu_milliseconds:18
evicted_keys:0                     #剔除(超过了maxmemory后)的key数量
keyspace_hits:0                    #命中次数
keyspace_misses:0                  #不命中次数
pubsub_channels:0                  #当前使用中的频道数量
pubsub_patterns:0                  #当前使用中的模式数量
latest_fork_usec:291               #最近一次fork操作的耗时，单位为微秒。
migrate_cached_sockets:0           #记录当前Redis正在进行migrate操作的目标Redis个数。例如Redis A分别向Redis B和C执行migrate操作，那么这个值就是2
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
tracking_total_keys:0
tracking_total_items:0
tracking_total_prefixes:0
unexpected_error_replies:0
total_reads_processed:18
total_writes_processed:17
io_threaded_reads_processed:0
io_threaded_writes_processed:0
  
# Replication
role:master                        #节点的角色（master|slave）
connected_slaves:1                 #连接的从节点个数
slave0:ip=10.1.1.11,port=6380,state=online,offset=1582,lag=1    #连接的从节点信息
master_replid:18879a7505f2a671c606a1f986cf31dad9389bb7
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0               #主节点偏移量
 
role:slave                         #节点的角色（master|slave）
master_host:10.234.116.60          #主节点IP
master_port:6379                   #主节点端口
master_link_status:up              #与主节点的连接状态
master_last_io_seconds_ago:6       #主节点最后与从节点的通信时间间隔，单位为秒
master_sync_in_progress:0          #从节点是否正在全量同步主节点RDB文件
slave_repl_offset:7632088          #复制偏移量
slave_priority:100                 #从节点优先级
slave_read_only:1                  #从节点是否只读
connected_slaves:0                 #连接从节点个数
master_replid:c42c9a846bc3c34133ed7a25d1ced7137e658042
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:7632088         #当前从节点作为其他节点的主节点时的复制偏移量
  
second_repl_offset:-1
repl_backlog_active:0              #复制缓冲区状态
repl_backlog_size:1048576          #复制缓冲区大小(单位字节)
repl_backlog_first_byte_offset:0   #复制缓冲区起始偏移量，标识当前缓冲区可用范围
repl_backlog_histlen:0             #标识复制缓冲区已存有效数据长度
  
# CPU
used_cpu_sys:0.807829              #Redis主进程在内核态所占用的CPU时钟总和
used_cpu_user:0.864519             #Redis主进程在用户态所占用的CPU时钟总和
used_cpu_sys_children:0.002514     #Redis子进程在内核态所占用的CPU时钟总和
used_cpu_user_children:0.002101    #Redis子进程在用户态所占用的CPU时钟总和
  
# Modules
  
# Commandstats（命令数据）
cmdstat_get:calls=2,usec=19,usec_per_call=9.50     #get命令调用总次数、总耗时、平均耗时(单位:微秒)
cmdstat_flushall:calls=1,usec=1036,usec_per_call=1036.00
cmdstat_debug:calls=3,usec=156,usec_per_call=52.00
cmdstat_set:calls=2,usec=31,usec_per_call=15.50    #set命令调用总次数、总耗时,平均耗时(单位:微秒)
cmdstat_auth:calls=2,usec=34,usec_per_call=17.00
cmdstat_command:calls=6,usec=4410,usec_per_call=735.00
cmdstat_monitor:calls=1,usec=2,usec_per_call=2.00
cmdstat_info:calls=2,usec=484,usec_per_call=242.00
cmdstat_keys:calls=3,usec=70,usec_per_call=23.33
  
# Cluster
cluster_enabled:0                  #节点是否为cluster模式。是1，否0
  
# Keyspace
db0:keys=26,expires=0,avg_ttl=0    #当前数据库key总数，带有过期时间的key总数，平均存活时间
10.1.1.11:6380> info
# Server
redis_version:6.0.16                      #Redis服务版本
redis_git_sha1:00000000                   #Git SHA
redis_git_dirty:0                         #Git dirty flag
redis_build_id:a016d5a8fa742e25           #Redis build id
redis_mode:standalone                     #运行模式，分为: Cluster、Sentinel、Standalone
os:Linux 3.10.0-1062.el7.x86_64 x86_64    #Redis所在机器的操作系统
arch_bits:64                              #架构(32或64位)
multiplexing_api:epoll                    #Redis所使用的事件处理机制
atomicvar_api:atomic-builtin
gcc_version:9.3.1                         #编译Redis时所使用的GCC版本
process_id:73309                          #Redis服务进程的PID
run_id:25a6bc791e9c33f9a4d526a9e08260a7661317c0    #Redis服务的标识符
tcp_port:6380                             #监听的端口
uptime_in_seconds:934                     #自Redis服务启动以来，运行的秒数
uptime_in_days:0                          #自Redis服务启动以来，运行的天数
hz:10                                     #serverCron每秒运行次数
configured_hz:10
lru_clock:16563116                        #以分钟为单位进行自增的时钟，用于LRU管理
executable:/apps/redis/bin/redis-server
config_file:/apps/redis/6379.conf         #Redis的配置文件
io_threads_active:0
  
# Clients
connected_clients:1                #当前客户端连接数
client_recent_max_input_buffer:8   #当前所有输入缓冲区中占用的最大容量。
client_recent_max_output_buffer:0  #当前所有输出缓冲区中占用的最大容量。
blocked_clients:0                  #正在等待阻塞命令(例如BLPOP等)的客户端数量
tracking_clients:0
clients_in_timeout_table:0
  
# Memory
used_memory:870272                 #Redis分配器分配的内存总量，也就是内部存储的所有数据内存占用量
used_memory_human:849.88K            #以可读的格式返回used_memory
used_memory_rss:8101888            #从操作系统的角度显示Redis进程占用的物理内存总量
used_memory_rss_human:7.73M          #以可读的格式返回used_memory_rss
used_memory_peak:925456            #内存使用的最大值，表示used_memory的峰值
used_memory_peak_human:903.77K       #以可读的格式返回used_memory_peak
used_memory_peak_perc:94.04%       #使用内存达到峰值内存的百分比，即(used_memory / used_memory_peak) *100%
used_memory_overhead:825224        #Redis为了维护数据集的内部机制所需的内存大小，包括所有客户端输出缓冲区、查询缓冲区、AOF重写缓冲区和主从复制的backlog。
used_memory_startup:803400         #Redis服务器启动时消耗的内存
used_memory_dataset:45048          #数据占用的内存大小，即used_memory - sed_memory_overhead
used_memory_dataset_perc:67.36%    #数据占用的内存大小的百分比，(used_memory_dataset / (used_memory - used_memory_startup)) * 100%
allocator_allocated:1146216
allocator_active:1503232
allocator_resident:3739648
total_system_memory:1019645952     #系统内存
total_system_memory_human:972.41M    #以可读的格式返回total_system_memory
used_memory_lua:37888              #Lua引警所消耗的内存大小
used_memory_lua_human:37.00K         #以可读的格式返回used_memory_lua
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0                        #Redis实例的最大内存配置
maxmemory_human:0B                   #以可读的格式返回
maxmemory_policy:noeviction        #当达到maxmemory时的淘汰策略
allocator_frag_ratio:1.31
allocator_frag_bytes:357016
allocator_rss_ratio:2.49
allocator_rss_bytes:2236416
rss_overhead_ratio:2.17
rss_overhead_bytes:4362240
mem_fragmentation_ratio:9.79       #used_memory_rss/used_memory比值，表示内存碎片率
mem_fragmentation_bytes:7274128
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:20496
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0       #Redis所使用的内存分配器。默认为jemalloc
active_defrag_running:0            #是否有活动的defrag任务正在运行，1表示有活动的defrag任务正在运行（defrag:表示内存碎片整理）
lazyfree_pending_objects:0         #0表示不存在延迟释放的挂起对象
  
# Persistence
loading:0                          #服务器是否正在载入持久化文件。是1，否0
rdb_changes_since_last_save:0      #自上次RDB后，Redis数据改动条数，即有多少个写入命令没有持久化
rdb_bgsave_in_progress:0           #bgsave操作是否正在进行中。是1，否0
rdb_last_save_time:1660730111      #上次成功生成RDB的时间戳（也可以使用lastsave命令获取）
rdb_last_bgsave_status:ok          #上次rdb持久化是否成功
rdb_last_bgsave_time_sec:0         #上次成功生成rdb文件耗时秒数
rdb_current_bgsave_time_sec:-1     #如果服务器正在创建rdb文件，那么这个记录的就是当前的创建操作已经耗费的秒数
rdb_last_cow_size:462848           #RDB过程中父进程与子进程相比执行了多少修改(包括读缓冲区，写缓冲区，数据修改等)。
aof_enabled:0                      #是否开启了AOF功能。是1，否0
aof_rewrite_in_progress:0          #标识AOF重写（bgrewriteaof）操作是否在进行中。是1，否0
aof_rewrite_scheduled:0            #AOF重写任务计划，当客户端发送bgrewriteaof指令，如果当前rewrite子进程正在执行，那么将客户端请求的bgrewriteaof变为计划任务，待aof子进程结束后执行bgrewriteaof
aof_last_rewrite_time_sec:-1       #上次AOF重写耗费的时长(单位是秒)
aof_current_rewrite_time_sec:-1    #如果AOF重写操作正在进行，则记录当前AOF重写所使用的时间(单位是秒)
aof_last_bgrewrite_status:ok       #上次AOF重写操作的状态
aof_last_write_status:ok           #上次AOF写磁盘的结果
aof_last_cow_size:0                #AOF过程中父进程与子进程相比执行了多少修改(包括读缓冲区，写缓冲区，数据修改等)。
module_fork_in_progress:0
module_fork_last_cow_size:0
  
# Stats
total_connections_received:1       #连接过的客户端总数
total_commands_processed:17        #执行过的命令总数
instantaneous_ops_per_sec:0        #每秒处理命令条数
total_net_input_bytes:1159         #输入总网络流量(以字节为单位)
total_net_output_bytes:23597       #输出总网络流量(以字节为单位)
instantaneous_input_kbps:0.00      #每秒输人字节数
instantaneous_output_kbps:0.00     #每秒输出字节数
rejected_connections:0             #拒绝的连接个数
sync_full:0                        #主从完全同步成功次数
sync_partial_ok:0                  #主从部分同步成功次数
sync_partial_err:0                 #主从部分同步失败次数
expired_keys:0                     #过期的key数量
expired_stale_perc:0.00
expired_time_cap_reached_count:0
expire_cycle_cpu_milliseconds:18
evicted_keys:0                     #剔除(超过了maxmemory后)的key数量
keyspace_hits:0                    #命中次数
keyspace_misses:0                  #不命中次数
pubsub_channels:0                  #当前使用中的频道数量
pubsub_patterns:0                  #当前使用中的模式数量
latest_fork_usec:291               #最近一次fork操作的耗时，单位为微秒。
migrate_cached_sockets:0           #记录当前Redis正在进行migrate操作的目标Redis个数。例如Redis A分别向Redis B和C执行migrate操作，那么这个值就是2
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
tracking_total_keys:0
tracking_total_items:0
tracking_total_prefixes:0
unexpected_error_replies:0
total_reads_processed:18
total_writes_processed:17
io_threaded_reads_processed:0
io_threaded_writes_processed:0
  
# Replication
role:master                        #节点的角色（master|slave）
connected_slaves:1                 #连接的从节点个数
slave0:ip=10.1.1.11,port=6380,state=online,offset=1582,lag=1    #连接的从节点信息
master_replid:18879a7505f2a671c606a1f986cf31dad9389bb7
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0               #主节点偏移量
 
role:slave                         #节点的角色（master|slave）
master_host:10.234.116.60          #主节点IP
master_port:6379                   #主节点端口
master_link_status:up              #与主节点的连接状态
master_last_io_seconds_ago:6       #主节点最后与从节点的通信时间间隔，单位为秒
master_sync_in_progress:0          #从节点是否正在全量同步主节点RDB文件
slave_repl_offset:7632088          #复制偏移量
slave_priority:100                 #从节点优先级
slave_read_only:1                  #从节点是否只读
connected_slaves:0                 #连接从节点个数
master_replid:c42c9a846bc3c34133ed7a25d1ced7137e658042
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:7632088         #当前从节点作为其他节点的主节点时的复制偏移量
  
second_repl_offset:-1
repl_backlog_active:0              #复制缓冲区状态
repl_backlog_size:1048576          #复制缓冲区大小(单位字节)
repl_backlog_first_byte_offset:0   #复制缓冲区起始偏移量，标识当前缓冲区可用范围
repl_backlog_histlen:0             #标识复制缓冲区已存有效数据长度
  
# CPU
used_cpu_sys:0.807829              #Redis主进程在内核态所占用的CPU时钟总和
used_cpu_user:0.864519             #Redis主进程在用户态所占用的CPU时钟总和
used_cpu_sys_children:0.002514     #Redis子进程在内核态所占用的CPU时钟总和
used_cpu_user_children:0.002101    #Redis子进程在用户态所占用的CPU时钟总和
  
# Modules
  
# Commandstats（命令数据）
cmdstat_get:calls=2,usec=19,usec_per_call=9.50     #get命令调用总次数、总耗时、平均耗时(单位:微秒)
cmdstat_flushall:calls=1,usec=1036,usec_per_call=1036.00
cmdstat_debug:calls=3,usec=156,usec_per_call=52.00
cmdstat_set:calls=2,usec=31,usec_per_call=15.50    #set命令调用总次数、总耗时,平均耗时(单位:微秒)
cmdstat_auth:calls=2,usec=34,usec_per_call=17.00
cmdstat_command:calls=6,usec=4410,usec_per_call=735.00
cmdstat_monitor:calls=1,usec=2,usec_per_call=2.00
cmdstat_info:calls=2,usec=484,usec_per_call=242.00
cmdstat_keys:calls=3,usec=70,usec_per_call=23.33
  
# Cluster
cluster_enabled:0                  #节点是否为cluster模式。是1，否0
  
# Keyspace
db0:keys=26,expires=0,avg_ttl=0    #当前数据库key总数，带有过期时间的key总数，平均存活时间

~~~

# standalone配置

~~~bash
###总体配置
daemonize yes                          #!是否是守护进程。默认值是no（yes|no）
port 6379                              #!redis实例监听的端口。默认值是6379（注意不同的实例监听的端口不一样）
 
loglevel notice                        #日志级别。默认值notice（debug|verbose|notice|warning），可以动态修改
logfile "/apps/redis/logs/redis_6379.log"    #!日志文件
dir /apps/redis/data_6379/             #!Redis工作目录(aof、rdb、日志文件都存放在此目录)。默认值是./ (当前目录)，可以动态修改
pidfile /apps/redis/run/redis_6379.pid #!Redis运行进程的pid文件。默认值是/var/run/redis.pid
# unixsocket /tmp/redis.sock           #指定套接字文件。默认值是空（不通过unix套接字来监听）
# unixsocketperm 700                   #unix套接字文件的权限。默认值是0
lua-time-limit 5000                    #Lua脚本”超时时间” ，单位是毫秒（超时不会真正停止脚本运行）。默认值是5000，可以动态修改
tcp-backlog 511                        #默认值511
activerehashing yes                    #指定是否激活重置哈希。默认值是yes，可以动态修改
databases 16                           #可用的数据库数，默认值是16
 
###安全相关配置
# bind 10.1.1.11                       #!Redis和哪个网卡进行绑定（注释掉bind表示监听ipv4和ipv6的所有地址）
# requirepass foobared                 #!客户端连接Redis的密码
 
###最大内存及策略
# maxmemory <bytes>                    #!Redis可以使用的最大内存。默认值是0(没有限制)，可以动态修改
# maxmemory-policy noeviction          #内存达到maxmemory上限时会触发相应的溢出控制策略。默认值是noeviction，可以动态修改
                                           #volatile-lru：用LRU算法删除过期的键值
                                           #allkeys-lru：用LRU算法删除所有键值
                                           #volatile-lfu：用LFU算法删除过期的键值
                                           #allkeys-lfu：用LFU算法删除所有键值
                                           #volatile-random：随机删除过期的键值
                                           #allkeys-random：随机删除任何键值
                                           #volatile-ttl：删除最近要到期的键值
                                           #noeviction：不删除键
# maxmemory-samples 5                  #检测LRU采样数。默认值是5，可以动态修改
 
###AOF相关配置
appendonly yes                         #!是否开启AOF持久化模式。默认值是no（no|yes），可以动态修改
appendfilename "appendonly.aof"        #!AOF文件的名称。默认值是appendonly.aof
appendfsync everysec                   #AOF缓冲区同步文件策略。默认值是everysec（always|everysec|no），可以动态修改
aof-load-truncated yes                 #加载AOF文件时，是否忽略AOF文件不完整的情况，让Redis正常启动。默认值是yes，可以动态修改
no-appendfsync-on-rewrite no           #在AOF重写期间是否不做fsync操作。设置为yes，表示在AOF重写期间不做fsync操作，暂时存在缓冲区中，等rewrite完成后再写入（不安全，redis挂掉时数据可能丢失）。默认值是no，可以动态修改
aof-rewrite-incremental-fsync yes      #每次批量写入硬盘数据量（默认为32MB），防止单次刷盘数据过多造成硬盘阻塞。默认值是yes，可以动态修改
auto-aof-rewrite-percentage 100        #触发重写AOF文件的增长比例条件，代表当前AOF文件的大小（aof_current_size）和上一次重写后AOF文件的大小（aof_base_size）的比值。默认值是100，可以动态修改
auto-aof-rewrite-min-size 64mb         #触发重写AOF文件的最小阀值(单位:MB)。默认值是64MB，可以动态修改
 
###RDB持久化
save 900 1                             #!900秒有1个写操作，就自动触发bgsave（如果没有save配置，代表不使用自动RDB策略）
save 300 10                            #!300秒有10个写操作，就自动触发bgsave
save 60 10000                          #!60秒有10000个写操作，就自动触发bgsave
dbfilename dump.rdb                    #!rdb文件的名称。默认值是dump.rdb，可以动态修改
rdbcompression yes                     #!是否压缩rdb文件。默认值是yes，可以动态修改
rdbchecksum yes                        #RDB文件是否使用校验和。默认值是yes，可以动态修改
stop-writes-on-bgsave-error yes        #bgsave执行错误，是否停止Redis接受写请求。默认值是yes，可以动态修改
 
###慢查询配置
slowlog-log-slower-than 10000          #慢查询被记录的阀值，单位是微秒（1秒=1000毫秒=1000000微秒），建议设置为1毫秒。默认值是10000，可以动态修改
slowlog-max-len 128                    #最多记录多少条慢查询，建议设置1000以上。默认值是128，可以动态修改
latency-monitor-threshold 0            #Redis服务内存延迟监控。默认值是0（关闭），可以动态修改
 
###客户端相关配置
# maxclients 10000                     #限制客户端最大连接数。默认值是10000，可以动态修改
timeout 0                              #限制客户端连接的最大空闲时间。默认值是0（不限制），可以动态修改
tcp-keepalive 0                        #检测TCP连接活性的周期(单位:秒)。默认值是0（不检测），可以动态修改
#客户端输出缓冲区限制：obl、oll、omem
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
#客户端输入缓冲区：qbuf、qbuf-free（动态调整）
 
###复制相关配置
# replicaof <masterip> <masterport>    #!指定当前从节点复制哪个主节点，参数是主节点的ip和port。不可以动态修改，但可以用slaveof命令设置
# masterauth <master-password>         #!如果主节点设置了requirepass，那么该选项就需要与主节点密码("requirepass")保持一致。可以动态修改
replica-read-only yes                  #!从节点是否开启只读模式（集群架构下从节点默认读写都不可用，需要调用readonly命令开启只读模式）。默认值是yes，可以动态修改
repl-disable-tcp-nodelay no            #!是否开启主从复制socket的NO_DELAY选项。设置为yes，Redis会合并小的TCP包来节省带宽，但是这样增加同步延迟，造成主从数据不一致。设置为no，主节点会立即发送同步数据，没有延迟。默认值是no，可以动态修改（同机架或同机房建议设置为no）
# min-replicas-to-write 3              #!需要至少3个从节点在线，延迟同步时间小于10秒，否则主节点可能停止接受写操作。可以动态修改
# min-replicas-max-lag 10
# repl-timeout 60                      #!RDB文件从创建到传输完毕消耗的总时间。如果总时间超过repl-timeout所配置的值，从节点将放弃接受RDB文件并清理已经下载的临时文件，导致全量复制失败。默认值是60s，可以动态修改
# repl-ping-replica-period 10          #主节点向从节点发送ping命令的周期，用于判定从节点是否存活(单位:秒)。默认值是10s，可以动态修改
# repl-backlog-size 1mb                #复制积压缓存区大小。默认值是1MB，可以动态修改
# repl-backlog-ttl 3600                #主节点在没有从节点的情况下多长时间后释放复制积压缓存区空间。默认值是3600，可以动态修改
replica-priority 100                   #从节点的优先级。默认值是100（1-100），可以动态修改
replica-serve-stale-data yes           #当从节点与主节点连接中断时，从节点是否继续处理客户端的请求。如果设置为"yes"，从节点可以继续处理客户端的请求。否则除info和slaveof命令之外，拒绝的所有请求并统一回复"SYNC with master in progress"。默认值是yes，可以动态修改
repl-diskless-sync no                  #是否开启无盘复制。默认值是no，可以动态修改
repl-diskless-sync-delay 5             #开启无盘复制后，需要延迟多少秒后进行创建RDB操作，一般用于同时加入多个从节点时，保证多个从节点可共享RDB。默认值是5，可以动态修改
 
###数据结构优化配置
hash-max-ziplist-entries 512           #哈希是否压缩，元素的个数。可以动态修改
hash-max-ziplist-value 64              #哈希是否压缩，元素的字节数。可以动态修改
list-max-ziplist-size -2               #列表是否压缩，8个字节。可以动态修改
set-max-intset-entries 512             #集合是否压缩，元素的个数。可以动态修改
zset-max-ziplist-entries 128           #有序集合是否压缩，元素的个数。可以动态修改
zset-max-ziplist-value 64              #有序集合是否压缩，元素的字节数。可以动态修改
hll-sparse-max-bytes 3000              #HyperLogLog数据结构优化参数。可以动态修改
~~~

# Sentinel配置

~~~bash
port 26379
daemonize yes
logfile "/opt/soft/redis/logs/26379.log"
dir /opt/soft/redis/data/
# sentinel monitor <master-name> <ip> <redis-port> <quorum>        #定义监控的主节点名、ip、port、主观下线票数。默认值是sentinel monitor mymaster 127.0.0.1 6379 2
# sentinel down-after-milliseconds <master-name> <milliseconds>    #Sentinel判定节点不可达的毫秒数。默认值是sentinel down-after-milliseconds mymaster 30000
# sentinel parallel-syncs <master-name> <numreplicas>              #在执行故障转移时，最多有多少个从服务器同时对新的主服务器进行同步。默认值是sentinel parallel-syncs mymaster 1
# sentinel failover-timeout <master-name> <milliseconds>           #故障迁移超时时间。默认值是sentinel failover-timeout mymaster 180000
# sentinel auth-pass <master-name> <password>                      #主节点密码
# sentinel notification-script <master-name> <script-path>         #故障转移期间脚本通知
# sentinel client-reconfig-script <master-name> <script-path>      #故障转移成功后脚本通知
~~~

# Cluster配置

~~~bash
# cluster-enabled yes                   #是否开启集群模式。
# cluster-config-file nodes-6379.conf   #集群配置文件名称
# cluster-node-timeout 15000            #集群节点超时时间(单位:毫秒)。默认值是15000，可以动态修改
# cluster-migration-barrier 1           #主从节点切换需要的从节点最小个数。默认值是1，可以动态修改
# cluster-replica-validity-factor 10    #从节点有效性判断因子，当从节点与主节点最后通信时间超过(cluster-node-timeout*slave-validity-factor)+repl-ping-slave-period 时，对应从节点不具备故障转移资格，防止断线时间过长的从节点进行故障转移。设置为0表示从节点永不过期。默认值是10，可以动态修改
# cluster-require-full-coverage yes     #集群是否需要所有的slot都分配给在线节点，才能正常访问。默认值是yes，可以动态修改
~~~








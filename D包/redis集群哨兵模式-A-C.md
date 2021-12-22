redis集群哨兵模式



#基础环境详见基础环境文档。

# 环境介绍

## 目标服务器信息

- A建信融通

  | ip            | 角色   | 端口       |
  | ------------- | ------ | ---------- |
  | 192.168.161.6 | master | 6379 26379 |
  | 192.168.161.7 | slave  | 6379 26379 |
  | 192.168.161.8 | slave  | 6379 26379 |

- C百融

  | ip             | hostname             | 角色   | 端口       |
  | -------------- | -------------------- | ------ | ---------- |
  | 192.168.161.12 | redis-rocketmq-node1 | master | 6379 26379 |
  | 192.168.161.51 | redis-rocketmq-node2 | slave  | 6379 26379 |
  | 192.168.161.52 | redis-rocketmq-node3 | slave  | 6379 26379 |

  

<font color=#DC143C>***注意:主从会随着服务的不稳定,主从角色会切换,2020/12/19日A包主节点为192.168.161.7</font>

​	

# 上传redis安装包至/data/install-tools/redis



- redis-5.0.10.bak.tar.gz为redis-5.0.10.tar.gz的编译之后的版本(生产无外网)



# 添加防护墙规则

```shell
#非root用户操作
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-port=26379/tcp --permanent
sudo firewall-cmd --zone=public --add-port=6379/tcp --permanent
sudo firewall-cmd --reload 
sudo firewall-cmd --zone=public --list-ports
```



# 更改主机名和hosts

```shell
#root用户操作

#node1修改主机名
echo redisnode1 > /etc/hostname && hostname $(cat /etc/hostname) && cat /etc/hostname
#node2修改主机名
echo redisnode2 > /etc/hostname && hostname $(cat /etc/hostname) && cat /etc/hostname
#node3修改主机名
echo redisnode3 > /etc/hostname && hostname $(cat /etc/hostname) && cat /etc/hostname

#三台主机修改hosts
vim /etc/hosts
192.168.161.6  redisnode1
192.168.161.7  redisnode2
192.168.161.8  redisnode3

```



# 修改系统参数：



```shell
#root用户操作
vi /etc/sysctl.conf
#添加一行：
vm.overcommit_memory = 1

#使系统参数生效
sysctl -p

#临时方法：
echo never > /sys/kernel/mm/transparent_hugepage/enabled 
#永久方法：
将上面一条命令写入rc.local文件
vi /etc/rc.d/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/enabled

#给/etc/rc.d/rc.local文件赋予执行权限
chmod 777 /etc/rc.d/rc.local
```



安装redis
===========



```
su - app
sudo mkdir -p /opt/redis && sudo chown -R app:app /opt/redis
mkdir -p /opt/redis/data && mkdir -p /opt/redis/app
mkdir -p /opt/redis/data/redis1/ && mkdir -p /opt/redis/data/redis  &&  mkdir -p /opt/redis/data/sentinel

cd /opt/redis/app
#上传安装包至当前目录
tar -zxvf redis-5.0.10.bak.tar.gz
mv redis-5.0.10.bak redis-5.0.10
```




# 更改配置文件



```shell

vi /opt/redis/data/redis/redis.conf

##############Master修改bind
#redis绑定的IP
bind 192.168.161.6
protected-mode yes
port 6379
requirepass ******    #******代表要设置的密码
masterauth ******    #******代表要设置的密码
maxmemory 2GB
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_1.pid
loglevel notice
logfile "/opt/redis/data/redis1/redis.log"
databases 16
always-show-logo yes
save 10 1
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /opt/redis/data/redis1/ 
# 修改以下两个参数作为slave节点(备注：158, 159注释去掉 改成master节点的IP )           
#replicaof 12.0.217.157 6379
#masterauth 123456
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly yes                
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes


##############Salve： 修改bind和replicaof
bind 192.168.161.7
protected-mode yes
port 6379
requirepass ******    #******代表要设置的密码
masterauth ******    #******代表要设置的密码
maxmemory 2GB
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_1.pid        
loglevel notice
logfile "/opt/redis/data/redis1/redis.log"    
databases 16
always-show-logo yes
save 10 1
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /opt/redis/data/redis1/ 
# 修改以下两个参数作为slave节点(备注：158, 159注释去掉 改成master节点的IP ) 
replicaof 192.168.161.6 6379
masterauth 123456
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly yes 
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```




# 启动redis



```shell
#非root启动redis:
/opt/redis/app/redis-5.0.10/src/redis-server /opt/redis/data/redis/redis.conf &
```



# 更改哨兵配置



```shell
#三个节点都需要修改sentinel.conf
vi /opt/redis/data/sentinel/sentinel.conf
#sentinel.conf内容：

port 26379
daemonize no
protected-mode no
pidfile /var/run/redis-sentinel.pid
logfile "/opt/redis/data/sentinel/sentinel.log"
dir /opt/redis/data/sentinel/
sentinel monitor jxrtMaster 192.168.161.6 6379 2  #填写master的地址
sentinel auth-pass jxrtMaster 123456
sentinel down-after-milliseconds jxrtMaster 30000
sentinel parallel-syncs jxrtMaster 1
sentinel failover-timeout jxrtMaster 180000
sentinel deny-scripts-reconfig yes

```



# 启动哨兵



```shell
非root启动sentinel:
/opt/redis/app/redis-5.0.10/src/redis-server /opt/redis/data/sentinel/sentinel.conf --sentinel &
```



# 查看集群状态



```shell
连接到主节点  能看到两个从节点
/opt/redis/app/redis-5.0.10/src/redis-cli -h 192.168.161.6 -p 6379
auth ******    #******代表要输入的密码
info replication 
#可以看到当前集群两个node的IP
```
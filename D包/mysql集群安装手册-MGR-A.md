# mysql集群安装手册-MGR

#基础环境详见基础环境文档。

## 服务器信息

| ip            | 角色   | 端口        |
| ------------- | ------ | ----------- |
| 192.168.161.3 | 主节点 | 3306、24801 |
| 192.168.161.4 | 从节点 | 3306、24801 |
| 192.168.161.5 | 从节点 | 3306、24801 |

## 关闭防火墙

三台机器都需要关闭

```bash
sudo systemctl status firewalld
```

## 上传安装包

上传以下文件到 192.168.161.3的 /data/install_packge目录下

my.cnf、mysql-shell-8.0.15-1.el7.x86_64.rpm 、mysql-5.7.32-linux-glibc2.12-x86_64.tar.gz

## 修改hosts文件

修改三台机器的hosts文件

```yaml
sudo vim /etc/hosts
#将以下内容写入文件末尾
192.168.161.3  mysql-01
192.168.161.4  mysql-02
192.168.161.5  mysql-03
```

## 修改hostname

修改三台机器的hostname

```yaml
#192.168.161.3
sudo hostnamectl set-hostname mysql-01
#192.168.161.4
sudo hostnamectl set-hostname mysql-02
#192.168.161.5
sudo hostnamectl set-hostname mysql-03
```

## 创建文件夹

在192.168.161.3执行以下命令

```
mkdir -p /data/mysql/data/
mkdir -p /data/mysql/app/
mkdir -p /data/mysql/conf/
mkdir -p /data/mysql/log/binlog/
mkdir -p /data/mysql/log/relaylog/
mkdir -p /data/mysql/mysqlrouter/
```

## 解压mysql安装包

在192.168.161.3执行以下命令

```
tar -zxvf /data/install_packge/mysql-5.7.32-linux-glibc2.12-x86_64.tar.gz -C /data/mysql/app/
```

## 复制配置文件

在192.168.161.3执行以下命令

```
cp /data/install_packge/my.cnf /data/mysql/conf/
```

## 创建itpuxdb-error.err

在192.168.161.3执行以下命令

```
touch /data/mysql/log/itpuxdb-error.err
```

## 分发解压包

在192.168.161.3执行以下命令

```
scp -r /data/mysql app@192.168.161.3:/data
scp -r /data/mysql app@192.168.161.4:/data
```

### 修改app用户环境变量

在三台机器上执行以下命令

```
vim /home/app/.bash_profile
#在文件末尾添加以下内容
PATH=$PATH:/data/mysql/app/mysql/bin:$HOME/bin
#重新加载环境变量
source /home/app/.bash_profile
```

## my.cnf

my.cnf内容

```yaml
[client]
port=24801
socket	= /data/mysql/data/mysql.sock

[mysql]
no-beep
prompt="\u@itpux \R:\m:\s [\d]> "
#no-auto-rehash
auto-rehash
default-character-set=utf8


[mysqld]
########basic settings########
#设置mysql的server id,每个mysql节点的id必须不一样
server-id=33306 
port=3306
user = app
#绑定mysql节点的IP
bind_address= 0.0.0.0   
basedir=/data/mysql/app/mysql
datadir=/data/mysql/data
socket=/data/mysql/data/mysql.sock
pid-file=/data/mysql/data/mysql.pid
character-set-server=utf8
skip-character-set-client-handshake=1
lower_case_table_names=1
autocommit = 0
#skip_name_resolve = 1
#mysql最大连接数
max_connections = 800   
max_connect_errors = 1000  
#mysql默认的引擎
default-storage-engine=INNODB   
transaction_isolation = READ-COMMITTED
explicit_defaults_for_timestamp = 1
sort_buffer_size = 32M
join_buffer_size = 128M
tmp_table_size = 72M
max_allowed_packet = 16M
sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER"
interactive_timeout = 1800
wait_timeout = 1800
read_buffer_size = 16M
read_rnd_buffer_size = 32M
#event_scheduler =1

query_cache_type = 1
query_cache_size=1M
table_open_cache=2000
thread_cache_size=768
myisam_max_sort_file_size=10G
myisam_sort_buffer_size=135M
key_buffer_size=32M
read_buffer_size=8M
read_rnd_buffer_size=4M

back_log=1024
#flush_time=0
open_files_limit=65536
table_definition_cache=1400
#binlog_row_event_max_size=8K
#sync_master_info=10000
#sync_relay_log=10000
#sync_relay_log_info=10000

########log settings########
log-output=FILE
general_log = 0
general_log_file=/data/mysql/log/itpuxdb-general.err
slow_query_log = ON
slow_query_log_file=/data/mysql/log/itpuxdb-query.err
long_query_time=10
log-error=/data/mysql/log/itpuxdb-error.err

log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
log_slow_slave_statements = 1
log_throttle_queries_not_using_indexes = 10
expire_logs_days = 90
min_examined_row_limit = 100


########innodb settings########
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
innodb_buffer_pool_size = 200M(80%)
innodb_buffer_pool_instances = 8
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1
innodb_lru_scan_depth = 2000
innodb_lock_wait_timeout = 5
#innodb_flush_method = O_DIRECT

innodb_log_file_size = 200M
innodb_log_files_in_group = 2 
innodb_log_buffer_size = 16M

innodb_undo_logs = 128
innodb_undo_tablespaces = 3
innodb_undo_log_truncate = 1
innodb_max_undo_log_size = 2G

innodb_flush_neighbors = 1
innodb_purge_threads = 4
innodb_large_prefix = 1
innodb_thread_concurrency = 64
innodb_print_all_deadlocks = 1
innodb_strict_mode = 1
innodb_sort_buffer_size = 64M
innodb_flush_log_at_trx_commit=1
innodb_autoextend_increment=64
innodb_concurrency_tickets=5000
innodb_old_blocks_time=1000
innodb_open_files=65536
innodb_stats_on_metadata=0
innodb_file_per_table=1
#innodb_checksum_algorithm=0
innodb_data_file_path=ibdata1:200M;ibdata2:200M;ibdata3:200M:autoextend:max:5G
innodb_temp_data_file_path = ibtmp1:200M:autoextend:max:20G

innodb_buffer_pool_dump_pct = 40
innodb_page_cleaners = 4
innodb_purge_rseg_truncate_frequency = 128
log_timestamps=system
#transaction_write_set_extraction=MURMUR32
show_compatibility_56=on
```

## 初始化mysql

在三台机器执行以下命令

```
mysqld  --defaults-file=/data/mysql/conf/my.cnf --initialize --user=app --basedir=/data/mysql/app/mysql --datadir=/data/mysql/data
```

## 启动mysql

在三台机器执行以下命令

```
mysqld_safe --defaults-file=/data/mysql/conf/my.cnf --user=app &
```

## 查看mysql初始密码

在三台机器执行以下命令 

```
#查看三台机器root密码
more /data/mysql/log/itpuxdb-error.err
#将密码记录
```

## 修改root密码

在三台机器执行以下命令

```
#登陆mysql 使用初始密码
mysql -uroot -h 127.0.0.1 -p
#修改密码  ******代表要设置的密码
set password=PASSWORD('******');  
#授权
grant all  privileges on *.* to 'root'@'%' identified by '******' with grant option;
#刷新
flush privileges;
```

## 修改my.conf

分别修改三台机器上的my.conf

server-id=33306 这个参数要三个节点修改成不同的值

注释掉这个配置#bind_address= 0.0.0.0

在my.conf文件末尾增加以下内容

```yaml
log_bin=/data/mysql/log/binlog/itpuxdb-binlog
log_bin_index=/data/mysql/log/binlog/itpuxdb-binlog.index
binlog_format=row
binlog_rows_query_log_events=on
binlog_checksum=none
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4
slave_preserve_commit_order=1

gtid_mode=on
enforce_gtid_consistency=1
log-slave-updates=1
binlog_gtid_simple_recovery=1

relay_log=/data/mysql/log/relaylog/itpuxdb-relay.log
relay-log-index=/data/mysql/log/relaylog/itpuxdb-relay.index
master_info_repository=table
relay_log_info_repository=table

plugin_load="group_replication=group_replication.so"

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="c7bf0f23-86cb-11ea-855f-000c29af7736"
loose-group_replication_start_on_boot=off
#loose-group_replication_local_address这个根据对应的主机IP修改 192.168.161.3节点的就写192.168.161.3，192.168.161.4就写192.168.161.4
loose-group_replication_local_address="192.168.161.3:24801"
loose-group_replication_group_seeds="192.168.161.3:24801,192.168.161.4:24801,192.168.161.5:24801" 
loose-group_replication_bootstap_group=off
#group_replication_single_primary_mode=off
#group_replication_enforce_update_everywhere_checks=on
group_replication_ip_whitelist="192.168.161.0/24"
```

## 重启mysql

三台机器执行以下命令

```
#停止
mysqladmin -uroot -p -h 127.0.0.1 -P 3306 shutdown
#启动
mysqld_safe --defaults-file=/data/mysql/conf/my.cnf --user=app &
```

## 配置mgr的第一个节点

192.168.161.3上执行

```yaml
#登陆
mysql -uroot -p -h 127.0.0.1 -P 3306
#创建用于复制的用户
set sql_log_bin=0;
create user zjzb@'%' identified by '******';  #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'%';
create user zjzb@'127.0.0.1' identified by '******';    #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'127.0.0.1';
create user zjzb@'localhost' identified by '******';     #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'localhost';
set sql_log_bin=1;
#配置复制所使用的用户
change master to master_user='zjzb', master_password='******' for channel 'group_replication_recovery';    #******代表要输入的密码
#安装mysql group replication这个插件。如果提示已安装则
install plugin group_replication soname 'group_replication.so'; 
#初始化复制组
set global group_replication_bootstrap_group=on;
start group_replication;
set global group_replication_bootstrap_group=off;
#查看当前组成员及状态如果状态正常则进行下面的步骤
select * from performance_schema.replication_group_members;
```

## 配置mgr的第二个节点

192.168.161.4上执行

```yaml
#登陆
mysql -uroot -p -h 127.0.0.1 -P 3306
#创建用于复制的用户
set sql_log_bin=0;
create user zjzb@'%' identified by '******';   #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'%';
create user zjzb@'127.0.0.1' identified by '******';   #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'127.0.0.1';
create user zjzb@'localhost' identified by '******';   #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'localhost';
set sql_log_bin=1;

配置复制所使用的用户
change master to master_user='zjzb', master_password='******' for channel 'group_replication_recovery';   #******代表要输入的密码
#安装mysql group replication这个插件。如果提示已安装则
install plugin group_replication soname 'group_replication.so'; 
加入前面创建好的复制组
start group_replication;
set global group_replication_bootstrap_group=off;
#查看当前组成员及状态如果状态正常则进行下面的步骤
select * from performance_schema.replication_group_members;
```

## 配置mgr的第三个节点

192.168.161.5上执行

```shell
#登陆
mysql -uroot -p -h 127.0.0.1 -P 3306
#创建用于复制的用户
set sql_log_bin=0;
create user zjzb@'%' identified by '******';     #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'%';
create user zjzb@'127.0.0.1' identified by '******';   #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'127.0.0.1';
create user zjzb@'localhost' identified by '******';    #******代表要输入的密码
grant replication slave,replication client on *.* to zjzb@'localhost';
set sql_log_bin=1;

配置复制所使用的用户
change master to master_user='zjzb', master_password='******' for channel 'group_replication_recovery';   #******代表要输入的密码
#安装mysql group replication这个插件。如果提示已安装则
install plugin group_replication soname 'group_replication.so'; 
加入前面创建好的复制组
start group_replication;
set global group_replication_bootstrap_group=off;
#查看当前组成员及状态如果状态正常则进行下面的步骤
select * from performance_schema.replication_group_members;
```

## 安装mysql shell

192.168.161.3上执行

```shell
#进入 mysql-shell-8.0.15-1.el7.x86_64.rpm 文件所在目录
cd /data/install_packge
#安装mysql shell
rpm -ivh mysql-shell-8.0.15-1.el7.x86_64.rpm
#进入mysql shell
mysqlsh --uri root@192.168.161.3:3306
#将groupreplication转换成innodbcluster
var cluster = dba.createCluster('dc',{adoptFromGR:true});
```

## 命令

```shell
#停止
mysqladmin -uroot -p -h 127.0.0.1 -P 3306 shutdown
#启动
mysqld_safe --defaults-file=/data/mysql/conf/my.cnf --user=app &
#查看当前组成员及状态
select * from performance_schema.replication_group_members;
```


mysql MGR方式初始化从库



#基础环境详见基础环境文档

# 环境介绍

## 目标服务器

### 预生产环境(MGR一主两从)

| ip             | hostname   | 角色   | 端口       |
| -------------- | ---------- | ------ | ---------- |
| 192.168.160.14 | mysqlnode1 | master | 3306 24801 |
| 192.168.160.15 | mysqlnode2 | slave  | 3306 24801 |
| 192.168.160.16 | mysqlnode3 | slave  | 3306 24801 |

### 生产环境(MGR一主两从)

| ip            | hostname | 角色   | 端口       |
| ------------- | -------- | ------ | ---------- |
| 192.168.161.3 | mysql-01 | master | 3306 24801 |
| 192.168.161.4 | mysql-02 | slave  | 3306 24801 |
| 192.168.161.5 | mysql-03 | slave  | 3306 24801 |



# 初始化从库步骤

## 主库master上执行



- 将主库数据全量导出,执行命令如下:

  ```shell
  mysqldump -uroot -h127.0.0.1 -p --all-databases --triggers --routines --events --master-data=2 > dbdump.db
  ```

  

## 从库两台机器上执行

- 通过客户端连接至mysql,执行命令如下:

  ```shell
  mysql -uroot -h127.0.0.1 -p
  #输入数据库密码
  ```

  

- 删除掉从节点上的所有业务相关的库,执行命令如下:

#格式: drop database `所有非系统库`, 如下:

```mysql
#mysql客户端
set global read_only=0;#从库默认是只读的,此语句为开启写权限
drop database `apolloconfigdb`;
drop database `apolloportaldb`;
drop database `cscscf`;
drop database `test`;
drop database `xxl-job`;
drop database `xxljob`;
drop database `zjzb`;
```

- 停止同步,并重设同步日志文件

```mysql
#mysql客户端
stop group_replication;
set global read_only=0;
reset master;  #删除所有的binglog日志文件，并将日志索引文件清空，重新开始所有新的日志文件。
reset slave;  #删除SLAVE数据库的relaylog日志文件，并重新启用新的relaylog文件；
exit;
```

- 从库数据导入

  ```shell
  mysql -h 127.0.0.1  -uroot -p  < /data/dbdump.db
  ```

- 通过客户端连接至mysql,执行命令如下:

  ```shell
  mysql -uroot -h127.0.0.1 -p
  #输入数据库密码
  ```

- 开启同步操作,执行命令如下:

```mysql
#mysql客户端
set sql_log_bin=0;
change master to master_user='zjzb', master_password='Zjzb123!@#' for channel 'group_replication_recovery';
set sql_log_bin=1;
start group_replication;
```

# 验证

## 主库写入

```mysql
#主库建库
create database test default character set utf8 collate utf8_general_ci;
#库中建表
CREATE TABLE comm_config (configId varchar(200) NOT NULL ,configValue varchar(1024) DEFAULT NULL ,description varchar(2000) DEFAULT NULL ,PRIMARY KEY (configId)) ENGINE=InnoDB DEFAULT CHARSET=utf8;
#表中插入数据
insert into comm_config(configId, configValue, description) values('name2', '架构与我2', '测试一下2');
commit
```

注意: 主库写入后要执行commit才生效

## 从库查看是否同步

```mysql
show database #查看是否有test数据库,有则成功,无则失败
use test
show tables  #查看从库中是否有表comm_config生成,有则成功,无则失败
select * from comm_config; #查看从库表中是否有数据生成,有则成功,无则失败
```

# 常用命令

```mysql
show master status\G
select @@read_only;
set global read_only=0;  #支持读写
set global read_only=1;  #只读
show variables like '%primary%';
select * from performance_schema.replication_group_members; #查看节点状态;三台online为正常
```

正常状态如图所示:

![avatar](images\mysql-MGR-node-status.png)


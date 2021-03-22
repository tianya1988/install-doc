# 环境介绍

#基础环境详见基础环境文档。

## 目标服务器信息

| ip             | hostname      | 端口 |
| -------------- | ------------- | ---- |
| 192.168.161.49 | mongo2-mysql1 | 3306 |
| 192.168.161.50 | mongo1-mysql2 | 3306 |

# 上传离线安装包至服务器/data/install-tools/mysql

# 安装(必须按顺序安装)



```shell
cd /data/install-tools/mysql 
sudo rpm -ivh mysql-community-common-5.7.32-1.el7.x86_64.rpm  --force --nodeps
sudo rpm -ivh mysql-community-libs-5.7.32-1.el7.x86_64.rpm --force --nodeps 
sudo rpm -ivh mysql-community-devel-5.7.32-1.el7.x86_64.rpm --force --nodeps
sudo rpm -ivh mysql-community-client-5.7.32-1.el7.x86_64.rpm --force --nodeps
sudo rpm -ivh mysql-community-server-5.7.32-1.el7.x86_64.rpm --force --nodeps

#启动mysql
sudo systemctl start mysqld 
#查看mysql运行状态
sudo systemctl status mysqld
#停止mysql 
sudo systemctl stop mysqld
#设置mysql开机启动
sudo systemctl enable mysqld

#查看随机密码
sudo cat /var/log/mysqld.log | grep password
```

 

# 创建mysql数据目录和日志文件



```shell
sudo mkdir -p /data/mysql/log/  /data/mysql/data  &&  sudo touch /data/mysql/log/mysqld.log
sudo systemctl stop mysqld
sudo cp -r /var/lib/mysql/* /data/mysql/data/
sudo chown -R mysql:mysql  /data/mysql
```



# 更改mysql的配置

- 192.168.161.49

  ```shell
  sudo vim /etc/my.cnf
  ```

  内容如下:

  ```
  [mysqld]
  server_id=49
  log-bin=mysql-bin
  datadir=/data/mysql/data
  socket=/var/lib/mysql/mysql.sock
  symbolic-links=0
  log-error=/data/mysql/log/mysqld.log
  pid-file=/var/run/mysqld/mysqld.pid
  lower_case_table_names = 1
  sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
  character-set-server=utf8
  
  [mysql]
  default-character-set=utf8
  ```

  

- 192.168.161.50

  ```shell
  sudo vim /etc/my.cnf
  ```

  内容如下:

  ```
  [mysqld]
  server_id=505
  datadir=/data/mysql/data
  socket=/var/lib/mysql/mysql.sock
  log_bin=mysql-bin
  symbolic-links=0
  replicate-do-db=smartdecision
  log-error=/data/mysql/log/mysqld.log
  pid-file=/var/run/mysqld/mysqld.pid
  lower_case_table_names = 1
  sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
  character-set-server=utf8
  
  [mysql]
  default-character-set=utf8
  ```

  

# 设置mysql管理员密码



```mysql
#启动mysql
sudo systemctl start mysqld
#连接mysql,输入刚才查看到的随机密码
mysql -uroot -p
#设置root密码  ******代表要设置的密码
set password=PASSWORD('******');
#配置可以远程登录
grant all  privileges on *.* to 'root'@'%' identified by '******' ;
#刷新权限
flush privileges;
```



# 添加防火墙规则



```shell
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```



# 配置主从



```mysql
#主服务器192.168.161.49  设置备份用户mysqnc的密码
GRANT REPLICATION SLAVE ON *.* to 'mysync'@'%' identified by '******';
flush privileges;
show master status;


#从服务器192.168.161.50; *master_log_pos=1133;需要在主节点的show master status\G查看，******代表mysync的密码
change master to master_host='YOUR-MASTER-IP',master_user='mysync',master_password='******',master_log_file='mysql-bin.000001',master_log_pos=1133;

start slave;

mysql -uroot -p -h IP

#从服务器查看状态
show slave status\G
一下两个进程正常运行，即YES状态
Slave_IO_Running: Yes
Slave_SQL_Running: Yes

#在主服务器创建一个testdb数据库；
create database testdb;

#连接到mysql从服务器

show databases ;
#可以看到从主服务器创建的testdb数据库
```

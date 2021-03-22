# Mongodb集群Replica Set模式部署文档


#基础环境详见基础环境文档。

## 安装机器

- 准生产环境

    | 序号 | IP地址         | 端口  |
    | ---- | -------------- | ----- |
    | 1    | 192.168.160.20 | 37017 |
    | 2    | 192.168.160.21 | 37017 |
    | 3    | 192.168.160.22 | 37017 |

- 生产环境

  | 序号 | IP地址         | 端口  |
  | ---- | -------------- | ----- |
  | 1    | 192.168.161.9  | 37017 |
  | 2    | 192.168.161.10 | 37017 |
  | 3    | 192.168.161.11 | 37017 |

  

## 配置用户权限

``` shell
# 切换到root用户
sudo su -
# 赋予app用户/opt和/data目录操作权限
chown -R app /opt
chown -R app /data
# 设置用户使用配置
vi /etc/security/limits.conf
# 将下面内容增加到limits.conf文件中
mongo  soft  nofile  64000
mongo  hard  nofile  64000
mongo  soft  nproc  32000
mongo  hard  nproc  32000
# 退出root
exit
```

## 移动mongo到/opt目录下

``` shell
# 拷贝 mongodb-linux-x86_64-rhel70-4.2.10.tgz 到 /data/install-tools 目录下
# 解压并移动mongo的安装文件到/opt目录并
tar -zxvf mongodb-linux-x86_64-rhel70-4.2.10.tgz -C /opt
# 修改目录名称
mv /opt/mongodb-linux-x86_64-rhel70-4.2.10 /opt/mongodb
```

## 创建mongo使用目录

``` shell
mkdir /data/mongodb 
cd /data/mongodb
mkdir conf data log pid
```

## 编辑mongo配置文件

cd /data/mongodb/conf

vim mongo.conf

``` shell
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  logRotate: rename
  path: /data/mongodb/log/mongod.log

# Where and how to store data.
storage:
  dbPath: /data/mongodb/data/
  journal:
    enabled: true
  directoryPerDB: true
#  engine:
#  mmapv1:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /data/mongodb/pid/mongod.pid  # location of pidfile

# network interfaces
net:
  port: 37017
  bindIp: 0.0.0.0  # Listen to local interface only, comment to listen on all interfaces.

#security:
  #keyFile:  /data/mongodb/conf/auth_rs.key

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 1000

#replication:
  #replSetName:  jxrtRS

#siharding:

## Enterprise-Only Options

#auditLog:

#snmp:
```

## 设置mongo环境变量

``` shell
# 切换到root用户, 不然没有权限修改
sudo su -
# 打开环境变量配置文件
vi /etc/profile
# 在文件最后添加下面内容
MONGO_HOME=/opt/mongodb
export PATH=$MONGO_HOME/bin:$PATH
# 重新加载配置文件使其生效
source /etc/profile
# 推出root用户
exit

# 启动测试
mongod --config /data/mongodb/conf/mongo.conf
# 连接服务
mongo 127.0.0.1:37017
# 如果不报错误的话，关闭服务
> use admin;
> db.shutdownServer();

输出信息如下：
2020-12-04T17:12:07.402+0800 I  NETWORK  [js] DBClientConnection failed to receive message from 127.0.0.1:37017 - HostUnreachable: Connection closed by peer
server should be down...
2020-12-04T17:12:07.404+0800 I  NETWORK  [js] trying reconnect to 127.0.0.1:37017 failed
2020-12-04T17:12:07.404+0800 I  NETWORK  [js] reconnect 127.0.0.1:37017 failed failed
```



## 集群配置

分别修改三台机器的mongo.conf文件

``` shell
# 打开配置文件
vim /data/mongodb/conf/mongo.conf
# 去掉配置文件中的replication部分注释
replication:
    replSetName:    jxrtRS
# 启动服务
mongod --config /data/mongodb/conf/mongo.conf
```

## 初始化集群

``` shell
# 关闭防火墙或开启三台机器的37017访问端口
# 方法一：
systemctl stop firewalld 
# 方法二：
firewall-cmd --zone=public --add-port=37017/tcp --permanent
firewall-cmd --reload 

# 在主节点运行以下命令初始化集群
mongo 127.0.0.1:37017

> rs.initiate( {
   _id : "jxrtRS",
   members: [
      { _id: 0, host: "192.168.160.20:37017" },
      { _id: 1, host: "192.168.160.21:37017" },
      { _id: 2, host: "192.168.160.22:37017" }
   ]
})

# 查看集群状态（备注：查看 health = 1）
> rs.status();

# 在primary节点上创建用户管理员账户：
> db.getSiblingDB("admin").createUser(
  {
    user: "root",
    pwd: "******",  #设置root的密码,******代表要设置的密码
    roles: [ { role: "root", db: "admin" } ]
  }
)

# 在primary节点上创建集群管理员账户：
> db.getSiblingDB("admin").createUser(
  {
    "user" : "rsadmin",
    "pwd" : "******",#设置rsadmin的密码,******代表要设置的密码
    roles: [ { "role" : "clusterAdmin", "db" : "admin" } ]
  }
)

# 创建应用账户
> db.getSiblingDB("admin").createUser(
  {
    "user" : "ccbscf_app",
    "pwd" : "******",#设置root的密码,******代表要设置的密码
    roles: [ { "role" : "readWriteAnyDatabase", "db" : "admin" } ]
  }
)



# 连接集群   ******代表要输入的密码
mongo --host jxrtRS/192.168.160.20:37017,192.168.160.21:37017,192.168.160.22:37017 -uccbscf_app -p ****** --authenticationDatabase admin
```

## 在配置文件中启用security

``` shell
# 启用keyfile参数
openssl rand -base64 756 > /data/mongodb/conf/auth_rs.key
chmod 400 /data/mongodb/conf/auth_rs.key
scp /data/mongodb/conf/auth_rs.key 192.168.160.21:/data/mongodb/conf/
scp /data/mongodb/conf/auth_rs.key 192.168.160.22:/data/mongodb/conf/

# 编辑三台机器的配置文件
vim /data/mongodb/conf/mongo.conf

# 去掉security前的# 注释
security:
  keyFile: /data/mongodb/conf/auth_rs.key

# 使用mongo客户端连接relica set进行测试  ******代表要输入的密码
mongo --host jxrtRS/192.168.160.20:37017,192.168.160.21:37017,192.168.160.22:37017 -uroot -p ****** --authenticationDatabase admin
```


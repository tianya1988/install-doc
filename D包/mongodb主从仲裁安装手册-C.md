# mongodb主从仲裁安装手册


#基础环境详见基础环境文档。

## 服务器信息

| ip             | 角色          | 端口  |
| -------------- | ------------- | ----- |
| 192.168.161.49 | 从(Secondary) | 27017 |
| 192.168.161.50 | 主（Primary） | 27017 |
| 192.168.161.13 | 仲裁(Arbiter) | 27017 |

<font color=#DC143C>***注意:主从会随着服务的不稳定,主从角色会切换,2020/12/18日主节点为192.168.161.49</font>

## 关闭三台主机防火墙

```shell
sudo systemctl stop firewalld
```

## 修改三台主机hostname

```shell
#192.168.161.50
sudo hostnamectl set-hostname mongo1-mysql2
#192.168.161.49
sudo hostnamectl set-hostname mongo2-mysql1
#192.168.161.13
sudo hostnamectl set-hostname mongo3-Arbiter
```

## 修改三台主机hosts文件

```
sudo vim /etc/hosts
```

文件末尾加入以下内容

```yaml
192.168.161.50	mongo1-mysql2
192.168.161.49	mongo2-mysql1
192.168.161.13	mongo3-Arbiter
```

## 安装

### 修改/data目录所属用户和用户组

```shell
sudo chown -R app:app /data
```

### 上传安装包到/data/install_packge

### 解压并重命名

```shell
tar -zxvf mongodb-linux-x86_64-rhel70-4.2.10.tgz
mv mongodb-linux-x86_64-rhel70-4.2.10 /data/mongodb
```

### 配置mongodb环境变量

```bash
sudo vim /etc/profile
```

在文件末尾加入以下内容

```yaml
#mongodb
export PATH=$PATH:/data/mongodb/bin
```

### 更新环境变量

```
source /etc/profile
```

### 创建文件夹

```shell
mkdir -p /data/mongodb/data
mkdir -p /data/mongodb/log
mkdir -p /data/mongodb/conf
mkdir -p /data/mongodb/key
```

### 在192.168.161.50上创建密钥文件

```shell
cd /data/mongodb/key
openssl rand -base64 741 >>keyfile
chmod 700 keyfile
```

### 将keyfile拷贝到另外两台机器

```shell
scp keyfile app@192.168.161.49:/data/mongodb/key
scp keyfile app@192.168.161.13:/data/mongodb/key
```

### 创建配置文件mongodb.conf

```shell
vim /data/mongodb/conf/mongodb.conf
```

主节点

```
pidfilepath=/data/mongodb/log/mongod.pid
#数据文件存放目录
dbpath = /data/mongodb/data
#日志文件存放目录
logpath =  /data/mongodb/log/mongodb.log
#端口 
port = 27017
#以守护程序的方式启用，即在后台运行
fork = true
#需要认证。如果放开注释，就必须创建MongoDB的账号，使用账号与密码才可远程访问，第一次安装建议注释 
#auth = true
#keyFile=/data/mongodb/key/keyfile
#允许远程访问，或者直接注释，127.0.0.1是只允许本地访问  0.0.0.0代表所有 
bind_ip = 0.0.0.0 
#表示复制集名称：rs0
replSet=rs0
```

从节点

```
pidfilepath=/data/mongodb/log/mongod.pid
#数据文件存放目录
dbpath = /data/mongodb/data
#日志文件存放目录
logpath =  /data/mongodb/log/mongodb.log
#端口 
port = 27017
#以守护程序的方式启用，即在后台运行
fork = true
#需要认证。如果放开注释，就必须创建MongoDB的账号，使用账号与密码才可远程访问，第一次安装建议注释 
#auth = true
#keyFile=/data/mongodb/key/keyfile
#允许远程访问，或者直接注释，127.0.0.1是只允许本地访问  0.0.0.0代表所有 
bind_ip = 0.0.0.0 
#表示复制集名称：rs0
replSet=rs0
```

仲裁节点

```
pidfilepath=/data/mongodb/log/mongod.pid
#数据文件存放目录
dbpath = /data/mongodb/data
#日志文件存放目录
logpath =  /data/mongodb/log/mongodb.log
#端口 
port = 27017
#以守护程序的方式启用，即在后台运行
fork = true
#需要认证。如果放开注释，就必须创建MongoDB的账号，使用账号与密码才可远程访问，第一次安装建议注释 
#auth = true
#keyFile=/data/mongodb/key/keyfile
#允许远程访问，或者直接注释，127.0.0.1是只允许本地访问  0.0.0.0代表所有 
bind_ip = 0.0.0.0 
#表示复制集名称：rs0
replSet=rs0
logappend=false
```

### 启动

进入mongo shell

```
mongo
```

主节点启动 192.168.161.50

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

从节点启动 192.168.161.49

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

仲裁节点启动

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

### 初始化复制集,192.168.161.50上执行

添加第一个成员

```
rs.initiate({_id: "rs0",members: [{ _id: 0 , host: "mongo1-mysql2:27017" }]})
```

添加第二个成员

```
rs.add("mongo2-mysql1:27017")
```

添加仲裁节点

```
rs.addArb("mongo3-Arbiter:27017")
```

### 创建用户

192.168.161.50节点执行

```
use admin
```

创建超级管理员

```
db.createUser({ 
    user: "admin", 
    pwd: "******",   #设置超级管理员密码,******代表要设置的密码
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] 
  } )
```

创建集群管理员

```
db.createUser({ 
    user: "rsadmin", 
    pwd: "******",  #设置集群管理员密码,******代表要设置的密码
    roles: [ { role: "clusterAdmin", db: "admin" } ] 
  } )
```

创建c包业务数据库和用户

```
use smartdecision
db.createUser({user:"TeamC",pwd :"******",roles:["readWrite"]})  #设置TeamC的密码,******代表要设置的密码
```

### 验证

登录从节点查看用户是否同步

```
#进入mongo shell
mongo
#切换到admin库
use admin
#查看当前库下的用户
show users
#切换到smartdecision库
use smartdecision
#查看当前库下的用户
show users
```

关闭主节点mongodb服务查看从节点是否切换为主节点

```
use admin
db.shutdownServer()
```

### 关闭集群启用认证参数

每个节点操作一致,可以先停掉从库和仲裁节点再停主库

```
use admin
db.shutdownServer()
```

修改配置文件启用认证

```
vim /data/mongodb/mongo
```

分别修改3个节点的配置文件,将之前注释的两行,启用

```
auth = true
keyFile=/data/mongodb/key/keyfile
```

启动数据库

主节点启动 192.168.161.50

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

从节点启动 192.168.161.49

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

仲裁节点启动 

```shell
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
```

登陆主库

```
#查看所有数据库 发现无输出
show dbs
#认证后查看数据库 可以看到数据库信息
use admin
db.auth('admin','******') #******代表要输入的密码
show dbs
```

### mongodb命令

```cmd
#启动
/data/mongodb/bin/mongod --config /data/mongodb/conf/mongodb.conf
#登录 不同用户权限不通
#进入mongo shell 如果找不到命令 source /etc/profile
mongo
#切换数据库
use admin
#认证 认证成功会返回 1
db.auth('admin','******') #数据库超级管理员密码
db.auth('rsadmin','******') #集群管理员密码
db.auth('TeamC','******') #应用数据库管理员,******代表要输入的密码
#停止 在admin库中执行(停止集群时先停从节点(SECONDARY)在停仲裁节点(ARBITER)最后停止主节点(PRIMARY))
db.shutdownServer()
#查看节点信息 主节点(PRIMARY)执行 
rs.status()
#查看复制集信息  仲裁节点(ARBITER)执行
db.isMaster()
#删库
db.dropDatabase()
#查看当前库下的用户
show users
#从库开启只读模式 从节点下执行
rs.secondaryOk()
#注意 主节点支持读写、从节点支持读但需要执行 rs.secondaryOk()
```

### mongodb相关目录

```
#安装目录
/data/mongodb
#日志目录
/data/mongodb/log
#数据目录
/data/mongodb/data
#秘钥文件目录
/data/mongodb/key
```


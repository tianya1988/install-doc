RocketMQ安装文档


#基础环境详见基础环境文档。



# 环境介绍

## 目标服务器信息

| ip             | hostname       | 角色 | 端口信息                            |
| -------------- | -------------- | ---- | ----------------------------------- |
| 192.168.161.33 | rocketmq-node1 | 主   | 9876 10911 10912(默认)  10909(默认) |
| 192.168.161.34 | rocketmq-node2 | 从   | 9876 10911 10912(默认)  10909(默认) |

# 安装

## 基础环境安装

### 配置hosts

```shell
sudo vim /etc/hosts
```

```
192.168.161.33    rocketmq-node1
192.168.161.34    rocketmq-node2
192.168.161.33    rocketmq-nameserver1
192.168.161.33    rocketmq-master1
192.168.161.34    rocketmq-nameserver2
192.168.161.34    rocketmq-master1-slave
```

### 暂时关闭防火墙

```shell
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo systemctl status firewalld
```

### 配置jdk环境

- 上传jdk-8u131-linux-x64.tar.gz至/data目录

- 解压并重命名

  ```shell
  tar -zxvf jdk-8u131-linux-x64.tar.gz
  mv jdk-8u131-linux-x64 java
  ```

- 编辑profile文件

  ```shell
  export JAVA_HOME=/data/java
  export PATH=$PATH:${JAVA_HOME}/bin
  ```

- source /etc/profile

- 删除系统中默认的jdk环境

  ```shell
  rm /usr/bin/java
  sudo ln -s  /data/java/bin/java  /usr/bin/java
  ```

  

## 安装步骤

### 上传依赖包至两台服务器

- 上传至服务器/data目录下

- 解压并重命名

  ```shell
  cd /data
  unzip rocketmq-all-4.7.1-bin-release.zip
  mv rocketmq-all-4.7.1-bin-release rocketmq
  ```

- 安装目录为/data/rocketmq


### 酌情修改broker占用内存

```shell
vim /data/rocketmq/bin/runbroker.sh
```

修改内容:

```shell
#根据环境调整,该环境为默认配置,未做修改
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

### 日志目录

- 创建日志目录

  ```sh
  mkdir -p /data/logs
  ```

- 替换日志目录配置:(把xml文件中的 '${user.home}' 替换为 '/data')

  ```shell
  sed -i  's#${user.home}#/data#g'  /data/rocketmq/conf/*.xml
  ```

### 创建数据存储目录

```
mkdir -p /data/mq-store
mkdir -p /data/mq-store/commitlog
mkdir -p /data/mq-store/consumequeue
mkdir -p /data/mq-store/index
mkdir -p /data/mq-store/abort
```

### 修改两台服务器的配置文件

- 主节点192.168.161.33

```shell
vim /data/rocketmq/conf/2m-2s-async/broker-a.properties
```

内容如下:

```
#给集群命名
brokerClusterName=pulian-rocketmq-cluster
#设置Broker节点名称,每个节点的名称必须不同
brokerName=broker-a
#设置Broker节点id,每个节点的id必须不同
brokerId=0
#设置NameServer地址
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#主体在一个broker上创建队列数量
defaultTopicQueueNums=4
#是否自动创建主题
autoCreateTopicEnable=true
#是否自动创建消费组
autoCreateSubscriptionGroup=true
#服务端监听端口
listenPort=10911
#每天什么时候执行删除过期文件,04表示凌晨4点
deleteWhen=04
#文件保留时间,168表示文件保留168个小时
fileReservedTime=168
#单个conmmitlog文件大小,1073741824表示1GB
mapedFileSizeCommitLog=1073741824
#单个consumequeue文件大小,300000表示单个Consumequeue文件中存储30W个ConsumeQueue条目
mapedFileSizeConsumeQueue=300000
#commitlog目录所在分区的最大使用比例，如果commitlog目录所在的分区使用比例大于该值，则触发过期文件删除
diskMaxUsedSpaceRatio=88
#broker存储目录 
storePathRootDir=/data/mq-store
#Commitlog存储目录
storePathCommitLog=/data/mq-store/commitlog
storePathConsumeQueue=/data/mq-store/consumequeue
storePathIndex=/data/mq-store/index
storeCheckpoint=/data/mq-store/checkpoint
abortFile=/data/mq-store/abort
#允许的最大消息体
maxMessageSize=65536
#broker角色,分为 ASYNC_MASTER SYNC_MASTER, SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式,默认为 ASYNC_FLUSH(异步刷盘),可选值SYNC_FLUSH(同步刷盘)
flushDiskType=ASYNC_FLUSH
```

- 从节点192.168.161.34

```shell
vim /data/rocketmq/conf/2m-2s-async/broker-a-s.properties
```

内容如下:

```
brokerClusterName=pulian-rocketmq-cluster
brokerName=broker-a
brokerId=1
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
defaultTopicQueueNums=4
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
listenPort=10911
deleteWhen=04
fileReservedTime=120
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=300000
diskMaxUsedSpaceRatio=88
storePathRootDir=/data/mq-store
storePathCommitLog=/data/mq-store/commitlog
storePathConsumeQueue=/data/mq-store/consumequeue
storePathIndex=/data/mq-store/index
storeCheckpoint=/data/mq-store/checkpoint
abortFile=/data/mq-store/abort
maxMessageSize=65536
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
```

### 启动服务

```shell
cd /data/rocketmq/bin
```

- 主节点192.168.161.33

  ```shell
  sh mqnamesrv >> mqnamesrv.log 2>&1 &
  nohup sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-a.properties >> mqbroker.log 2>&1 &
  ```

- 从节点192.168.161.34

  ```
  sh mqnamesrv >> mqnamesrv.log 2>&1 &
  nohup sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-a-s.properties >> mqbroker.log 2>&1 &
  ```

### 验证

两台服务器上都执行:

#### 验证nameserver

```shell
ps -ef | grep mqnamesrv
netstat -ntlp | grep 9876
```

#### 验证broker

```shell
ps -ef | grep broker
netstat -ntlp | grep 10911
```



### 关闭服务命令

#### 关闭broker

```shell
sh mqshutdown broker
```

#### 关闭nameserver

```shell
sh mqshutdown namesrv
```


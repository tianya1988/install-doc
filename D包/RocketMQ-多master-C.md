**RocketMQ多master安装手册**



#基础环境详见基础环境文档。



# 环境介绍

## 目标服务器信息

| ip             | 角色 | 端口信息                            |  |
| -------------- | -------------- | ---- | ----------------------------------- |
| 192.168.161.12 | 主   | 9876 10911 10912(默认)  10909(默认) |  |
| 192.168.161.51 | 主 | 9876 10911 10912(默认)  10909(默认) |  |
| 192.168.161.52 | 主 | 9876 10911 10912(默认)  10909(默认) |  |

# 安装

## 基础环境安装

### 配置hosts

```shell
sudo vim /etc/hosts
```

```
192.168.161.12    rocketmq-broker-m1
192.168.161.51    rocketmq-broker-m2
192.168.161.52    rocketmq-broker-m3
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

### 修改三台服务器的配置文件

- 主节点192.168.161.12

```shell
vim /data/rocketmq/conf/2m-noslave/broker-a.properties
```

内容如下:

```shell
brokerClusterName=rocketmq-3m-cluster
brokerName=rocketmq-broker-m1
brokerId=0
namesrvAddr=192.168.161.12:9876;192.168.161.51:9876;192.168.161.52:9876
defaultTopicQueueNums=6
messageIndexSafe=true
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
waitTimeMillsInSendQueue=5000
listenPort=10911
deleteWhen=04
fileReservedTime=168
#单个conmmitlog文件大小默认1GB
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=6000000
#commitlog目录所在分区的最大使用比例，如果commitlog目录所在的分区使用比例大于该值，则触发过期文件删除
diskMaxUsedSpaceRatio=88
#默认允许的最大消息体默认4M,4194304
#maxMessageSize=65536
sendMessageThreadPoolNums=4
pullMessageThreadPoolNums=4
useReentrantLockWhenPutMessage=true
defaultReadQueueNums=16
defaultWriteQueueNums=16
storePathRootDir=/data/mq-store
storePathCommitLog=/data/mq-store/commitlog
storePathConsumeQueue=/data/mq-store/consumequeue
storePathIndex=/data/mq-store/index
storeCheckpoint=/data/mq-store/checkpoint
abortFile=/data/mq-store/abort
brokerRole=ASYNC_MASTER
#刷盘方式,默认为 ASYNC_FLUSH(异步刷盘),可选值SYNC_FLUSH(同步刷盘)
flushDiskType=ASYNC_FLUSH
```

- 主节点192.168.161.51

```shell
vim /data/rocketmq/conf/2m-noslave/broker-b.properties
```

内容如下:

```sh
brokerClusterName=rocketmq-3m-cluster
brokerName=rocketmq-broker-m2
brokerId=0
namesrvAddr=192.168.161.12:9876;192.168.161.51:9876;192.168.161.52:9876
defaultTopicQueueNums=6
messageIndexSafe=true
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
waitTimeMillsInSendQueue=5000
listenPort=10911
deleteWhen=04
fileReservedTime=168
#单个conmmitlog文件大小默认1GB
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=6000000
#commitlog目录所在分区的最大使用比例，如果commitlog目录所在的分区使用比例大于该值，则触发过期文件删除
diskMaxUsedSpaceRatio=88
#默认允许的最大消息体默认4M,4194304
#maxMessageSize=65536
sendMessageThreadPoolNums=4
pullMessageThreadPoolNums=4
useReentrantLockWhenPutMessage=true
defaultReadQueueNums=16
defaultWriteQueueNums=16
storePathRootDir=/data/mq-store
storePathCommitLog=/data/mq-store/commitlog
storePathConsumeQueue=/data/mq-store/consumequeue
storePathIndex=/data/mq-store/index
storeCheckpoint=/data/mq-store/checkpoint
abortFile=/data/mq-store/abort
brokerRole=ASYNC_MASTER
#刷盘方式,默认为 ASYNC_FLUSH(异步刷盘),可选值SYNC_FLUSH(同步刷盘)
flushDiskType=ASYNC_FLUSH
```


- 主节点192.168.161.52

```shell
vim /data/rocketmq/conf/2m-noslave/broker-c.properties
```

内容如下:

```sh
brokerClusterName=rocketmq-3m-cluster
brokerName=rocketmq-broker-m3
brokerId=0
namesrvAddr=192.168.161.12:9876;192.168.161.51:9876;192.168.161.52:9876
defaultTopicQueueNums=6
messageIndexSafe=true
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
waitTimeMillsInSendQueue=5000
listenPort=10911
deleteWhen=04
fileReservedTime=168
#单个conmmitlog文件大小默认1GB
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=6000000
#commitlog目录所在分区的最大使用比例，如果commitlog目录所在的分区使用比例大于该值，则触发过期文件删除
diskMaxUsedSpaceRatio=88
#默认允许的最大消息体默认4M,4194304
#maxMessageSize=65536
sendMessageThreadPoolNums=4
pullMessageThreadPoolNums=4
useReentrantLockWhenPutMessage=true
defaultReadQueueNums=16
defaultWriteQueueNums=16
storePathRootDir=/data/mq-store
storePathCommitLog=/data/mq-store/commitlog
storePathConsumeQueue=/data/mq-store/consumequeue
storePathIndex=/data/mq-store/index
storeCheckpoint=/data/mq-store/checkpoint
abortFile=/data/mq-store/abort
brokerRole=ASYNC_MASTER
#刷盘方式,默认为 ASYNC_FLUSH(异步刷盘),可选值SYNC_FLUSH(同步刷盘)
flushDiskType=ASYNC_FLUSH
```

### 启动服务

```shell
cd /data/rocketmq/bin
```

- 主节点192.168.161.12

```shell
nohup sh mqnamesrv >> mqnamesrv.log 2>&1 &
nohup sh mqbroker -c /data/rocketmq/conf/2m-noslave/broker-a.properties >> mqbroker.log 2>&1 &
```

- 主节点192.168.161.51

```shell
nohup sh mqnamesrv >> mqnamesrv.log 2>&1 &
nohup sh mqbroker -c /data/rocketmq/conf/2m-noslave/broker-b.properties >> mqbroker.log 2>&1 &
```

- 主节点192.168.161.52

```shell
nohup sh mqnamesrv >> mqnamesrv.log 2>&1 &
nohup sh mqbroker -c /data/rocketmq/conf/2m-noslave/broker-c.properties >> mqbroker.log 2>&1 &
```



### 验证

三台服务器上都执行:

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

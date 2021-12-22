Elasticsearch安装手册



[TOC]



# 环境介绍

## 目标服务器信息

| ip             | hostname | 角色        | 端口      |
| :------------- | -------- | ----------- | --------- |
| 192.168.161.40 | node1    | master data | 9200 9300 |
| 192.168.161.41 | node2    | master data | 9200 9300 |
| 192.168.161.42 | node3    | master data | 9200 9300 |



# 基础环境搭建



## 暂时关闭防火墙,服务搭建好后开启

```shell
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
#不想重启系统，使用命令setenforce 0
sudo setenforce 0
```

	检查防火墙状态

```shell
sudo systemctl status firewalld.service
sudo getenforce   #Disabled为正确
```

## 配置host

```shell
sudo vim /etc/hosts
```

内容如下:

```
192.168.161.40  node1
192.168.161.41  node2
192.168.161.42  node3
```

## 配置jdk环境

- 上传安装包到指定目录

- 解压安装包并重命名,最终安装目录为/data/java

- 编辑/etc/profile文件,添加如下信息

  ```shell
  export JAVA_HOME=/data/java
  export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
  export PATH=$PATH:$JAVA_HOME/bin
  ```
  
- 加载环境变量: source /etc/profile
- 去掉系统默认jdk环境: rm /usr/bin/java
- sudo ln -s  /data/java/bin/java  /usr/bin/java

## 检查NTP服务是否正常

执行命令: ntpq -p  , 输出以下内容为正常

```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*192.168.2.91    185.255.55.20    3 u  418 1024  377    1.206    1.033   0.799
 LOCAL(0)        .LOCL.          10 l  12d   64    0    0.000    0.000   0.000
```

## 修改系统参数

```shell
sudo echo '* soft nofile 655360'>> /etc/security/limits.conf
sudo echo '* hard nofile 655360'>> /etc/security/limits.conf
sudo echo '* hard nproc  40960'>> /etc/security/limits.conf
sudo echo '* soft nproc  40960'>> /etc/security/limits.conf
sudo echo 'app soft memlock unlimited' >> /etc/security/limits.conf
sudo echo 'app hard memlock unlimited' >> /etc/security/limits.conf
sudo echo 'vm.swappiness=1' >> /etc/sysctl.conf
sudo echo 'vm.max_map_count=262144' >> /etc/sysctl.conf
#sysctl参数生效
sysctl -p

#用户退出再登录limits生效,(如果不生效,重启系统)
```



# elasticsearch安装

## 各节点创建依赖目录

```shell
mkdir -p /data/es-data/data
mkdir -p /data/es-data/logs
```

## 各节点修改jvm参数,建议为系统内存的一半

```shell
vim /data/elasticsearch/config/jvm.options

-Xms16g
-Xmx16g
```

## 各节点修改配置文件

- 192.168.161.40  node1 上

  ```shell
  vim /data/elasticsearch/config/elasticsearch.yml
  ```

  内容如下:

  ```shell
  cluster.name: zjzb-pl-cluster
  node.name: node1
  node.master: true
  node.data: true
  node.max_local_storage_nodes: 3
  path.data: /data/es-data/data
  path.logs: /data/es-data/logs
  network.host: node1
  http.port: 9200
  transport.tcp.port: 9300
  cluster.initial_master_nodes: ["node1"]
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  cluster.routing.allocation.same_shard.host: true
  bootstrap.memory_lock: true
  gateway.recover_after_master_nodes: 2
  gateway.recover_after_data_nodes: 2
  discovery.zen.ping.unicast.hosts: ["node1", "node2","node3"]
  ```

  

- 192.168.161.41  node2 上

  ```shell
  vim /data/elasticsearch/config/elasticsearch.yml
  ```

  内容如下:

  ```shell
  cluster.name: zjzb-pl-cluster
  node.name: node2
  node.master: true
  node.data: true
  node.max_local_storage_nodes: 3
  path.data: /data/es-data/data
  path.logs: /data/es-data/logs
  network.host: node2
  http.port: 9200
  transport.tcp.port: 9300
  cluster.initial_master_nodes: ["node1"]
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  cluster.routing.allocation.same_shard.host: true
  bootstrap.memory_lock: true
  gateway.recover_after_master_nodes: 2
  gateway.recover_after_data_nodes: 2
  discovery.zen.ping.unicast.hosts: ["node1", "node2","node3"]
  ```

  - 192.168.161.42  node3 上

  ```shell
  vim /data/elasticsearch/config/elasticsearch.yml
  ```

  内容如下:

  ```shell
  cluster.name: zjzb-pl-cluster
  node.name: node3
  node.master: true
  node.data: true
  node.max_local_storage_nodes: 3
  path.data: /data/es-data/data
  path.logs: /data/es-data/logs
  network.host: node3
  http.port: 9200
  transport.tcp.port: 9300
  cluster.initial_master_nodes: ["node1"]
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  cluster.routing.allocation.same_shard.host: true
  bootstrap.memory_lock: true
  gateway.recover_after_master_nodes: 2
  gateway.recover_after_data_nodes: 2
  discovery.zen.ping.unicast.hosts: ["node1", "node2","node3"]
  ```

## 启动

执行命令:

```
/data/elasticsearch/bin/elasticsearch -d
```

## 验证

任意节点执行:

```shell
curl 192.168.161.40:9200/_cat/health?v

#返回信息如下,green为健康状态
1607953313 13:41:53 zjzb-pl-cluster green 3 3 0 0 0 0 0 0 - 100.0%
```


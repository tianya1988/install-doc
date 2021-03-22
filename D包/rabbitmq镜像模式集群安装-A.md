rabbitmq镜像模式集群安装


#基础环境详见基础环境文档。

# 环境介绍

## 目标服务器信息

| ip             | host      | 角色   | 端口                  |
| -------------- | --------- | ------ | --------------------- |
| 192.168.161.14 | rabbitmq1 | master | 4369 5672 15672 25672 |
| 192.168.161.15 | rabbitmq2 | master | 4369 5672 15672 25672 |
| 192.168.161.16 | rabbitmq3 | master | 4369 5672 15672 25672 |



# 上传安装文件至/data/install-tools/rabbitmq目录下



# 更改主机名

```shell
root用户操作
三台服务器的主机名分别设置成rabbitmq1、rabbitmq2、rabbitmq3

三台服务器分别配置hosts
vim /etc/hosts
192.168.161.14  rabbitmq1
192.168.161.15  rabbitmq2
192.168.161.16  rabbitmq3
```



# 安装rabbitmq依赖和rabbitmq



```shell
cd /data/install-tools/rabbit

sudo rpm -ivh erlang-19.0.4-1.el7.centos.x86_64.rpm 
sudo rpm -ivh socat-1.7.3.2-2.el7.x86_64.rpm 
xz -d rabbitmq-server-generic-unix-3.6.12.tar.xz 
tar -xvf rabbitmq-server-generic-unix-3.6.12.tar

#把rabbitmq目录放到/opt下
sudo cp -r /data/install-tools/rabbitmq_server-3.6.12 /opt 
sudo chown -R app:app /opt/rabbitmq_server-3.6.12

```



# 添加环境变量



```shell
#编辑 /etc/profile
sudo vim /etc/profile

#在末尾加入以下配置
export PATH=$PATH:/opt/rabbitmq_server-3.6.12/sbin

#更新环境变量
source /etc/profile
```



# rabbitmq基本操作



```shell
rabbitmq用非root用户启动

#启动：
rabbitmq-server -detached

#关闭：
rabbitmqctl stop

#查看状态：
rabbitmqctl status

#设置admin,******代表要设置的账户密码：
rabbitmqctl add_user admin ******

#更改账户名密码：
rabbitmqctl change_password 用户名 密码

#设置admin为管理员权限：
rabbitmqctl set_user_tags admin administrator 

#删除guest用户：
rabbitmqctl delete_user guest

#列出用户： 
rabbitmqctl list_users

#删除用户： 
rabbitmqctl delete_user 用户名

#赋给admin用户所有权限：
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

#列出权限：
rabbitmqctl list_permissions

#启用rabbitmq网页管理插件：
rabbitmq-plugins enable rabbitmq_management

#列出启用的管理插件：
rabbitmq-plugins list

#访问管理页面：http://ip:15672 端口默认为15672

```



# 添加防火墙规则



```shell
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-port=4369/tcp --permanent && sudo firewall-cmd --zone=public --add-port=5672/tcp --permanent && sudo firewall-cmd --zone=public --add-port=15672/tcp --permanent && sudo firewall-cmd --zone=public --add-port=25672/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-ports
```



# 集群配置



```shell
#在rabbitmq2和rabbitmq3上分别停止rabbitmq： 
rabbitmqctl stop

#同步master的'.erlang.cookie'文件到rabbitmq2和rabbitmq3上,'.erlang.cookie'位于用户家目录下
scp /home/app/.erlang.cookie app@192.168.161.15:/home/app/

#在rabbitmq2和rabbitmq3上分别启动rabbitmq： 
rabbitmq-server -detached

#在rabbitmq2和rabbitmq3上分别停止应用： 
rabbitmqctl stop_app

#在rabbitmq2和rabbitmq3上分别将master加入到集群： 
rabbitmqctl join_cluster rabbit@rabbitmq1

#在rabbitmq2和rabbitmq3上分别启动应用： 
rabbitmqctl start_app

#查看集群状态： 
rabbitmqctl cluster_status

#如果需要移除集群节点，执行下面命令：
rabbitmqctl forget_cluster_node rabbit@rabbitmq2(具体节点名称)

#修改集群名称（任意一个节点操作，默认为master node名称）
rabbitmqctl set_cluster_name rabbitmq_zjzb

#设置镜像队列策略(任意一个节点操作)
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

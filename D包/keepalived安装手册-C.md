keepalived安装手册



#基础环境详见基础环境文档。

# keepalived安装手册

## 服务器信息

| ip             | 角色 |
| -------------- | ---- |
| 192.168.161.45 | 主   |
| 192.168.161.46 | 备   |



配置生成的虚拟ip

| ip              | 角色                              |
| --------------- | --------------------------------- |
| 192.168.161.102 | 虚拟ip(keepalived.conf文件中指定) |



## 上传安装包到/data/install-packages

## 安装依赖

```shell
sodo yum install gcc-c++
sodo yum install pcre*
sodo yum install openssl*
```

## 解压

```shell
tar -zxvf keepalived-2.1.5.tar.gz
```

## 进入文件夹keepalived-2.1.5

```shell
cd keepalived-2.1.5
```

## 目录授权

```shell
sudo chmod -R 753 /usr/lib/systemd/
```

## 配置keepalived

```shell
./configure --prefix=/data/keepalived
```

## 编译安装

```shell
make && make install
```

## 创建keepalived配置文件目录

```shell
sudo mkdir -p /etc/keepalived	
```

## 复制配置文件

```sh
cp /data/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

## 修改配置文件

### 主节点192.168.161.45

```shell
vim /etc/keepalived/keepalived.conf
```

写入以下内容

```yaml
! Configuration File for keepalived

global_defs {
  #路由id，主备节点不能相同
  router_id node1
}

#自定义监控脚本
vrrp_script chk_haproxy {
  # 脚本位置;如果nginx没有运行,则systemctl stop keepalived
  script "/etc/keepalived/nginx_check.sh"
  # 脚本执行的时间间隔
  interval 1
  weight 0
  
}

vrrp_instance VI_1 {
    # Keepalived的角色，MASTER 表示主节点，BACKUP 表示备份节点
    state MASTER
    # 指定监测的网卡，可以使用 ifconfig 或 ip a 进行查看
    interface eth0
    # 虚拟路由的id，主备节点需要设置为相同
    virtual_router_id 2
    # 优先级，主节点的优先级需要设置比备份节点高
    priority 100
    #当前机器ip
    unicast_src_ip  192.168.161.45
    #另一个keepalived节点ip
    unicast_peer {
        192.168.161.46
    }
    # 设置主备之间的检查时间，单位为秒
    advert_int 1
    # 定义验证类型和密码
    authentication {
        auth_type PASS
        auth_pass 1111
    }
  
    # 虚拟IP地址，可以设置多个
    virtual_ipaddress {
        192.168.161.102
    }
    
   # 调用上面自定义的监控脚本
    track_script {
       chk_haproxy
    }

}
```

### 从节点192.168.161.46

```shell
vim /etc/keepalived/keepalived.conf
```

写入以下内容

```yaml
! Configuration File for keepalived

global_defs {
  #路由id，主备节点不能相同
  router_id node2
}

#自定义监控脚本
vrrp_script chk_haproxy {
  # 脚本位置
  script "/etc/keepalived/nginx_check.sh"
  # 脚本执行的时间间隔
  interval 1
  weight 0
  
}

vrrp_instance VI_1 {
    # Keepalived的角色，MASTER 表示主节点，BACKUP 表示备份节点
    state BACKUP
    # 指定监测的网卡，可以使用 ifconfig 或 ip a 进行查看
    interface eth0
    # 虚拟路由的id，主备节点需要设置为相同
    virtual_router_id 2
    # 优先级，主节点的优先级需要设置比备份节点高
    priority 90
    #当前机器ip
    unicast_src_ip  192.168.161.46
    #另一个keepalived节点ip
    unicast_peer {              
        192.168.161.45
    } 
   # 设置主备之间的检查时间，单位为秒
    advert_int 1
    # 定义验证类型和密码
    authentication {
        auth_type PASS
        auth_pass ****  #设置验证密码
    }
      
    # 虚拟IP地址，可以设置多个
    virtual_ipaddress {
        192.168.161.102
    }
    
   # 调用上面自定义的监控脚本
    track_script {
       chk_haproxy
    }

}
```

## 编写nginx监控脚本

两个节点都需要编写

```shell
vim /etc/keepalived/nginx_check.sh
```

写入以下内容

```shell
#! /bin/bash
#ng_num代表nginx运行的个数
ng_num=`ps -C nginx --no-header |wc -l`
#echo $ng_num
if [ $ng_num -eq 0 ];then
    echo 'nginx not running, stop keepalived!'
    sudo systemctl stop keepalived
fi
```

修改nginx_check.sh为可执行文件

```shell
chmod 777 nginx_check.sh 
#chmod +x nginx_check.sh  也可以
```

## keepalived命令

```shell
 #启动（这里要先保证nginx是已经启动的，不然keepalived启动了以后会执行脚本发现nginx没启动会运行脚本停止掉keepalived）
 sudo service keepalived start
 #停止
 sudo service keepalived stop
 #查看状态
 sudo service keepalived status 
```

## 验证keepalived

主从节点都执行观察网卡信息发现vip 192.168.161.102 在主节点上

```shell
ip a
```

主节点网卡信息

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0c:da:41:1d:3e:37 brd ff:ff:ff:ff:ff:ff
    inet 192.168.161.45/24 brd 192.168.161.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.161.102/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::eda:41ff:fe1d:3e37/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:f9:35:f3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:f9:35:f3 brd ff:ff:ff:ff:ff:ff
```

从节点网卡信息

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0c:da:41:1d:9b:e3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.161.46/24 brd 192.168.161.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::eda:41ff:fe1d:9be3/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:f9:35:f3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:f9:35:f3 brd ff:ff:ff:ff:ff:ff
```

## vip漂移验证

关闭主节点nginx查看从节点网卡信息 如果vip漂移到从节点则成功

恢复主节点，vip被主机夺回，转向主机
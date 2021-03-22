**k8s安装文档-A**



#基础环境详见基础环境文档。

# 环境介绍

## 目标服务器信息

| ip             | hostname           | 角色   |
| :------------- | ------------------ | ------ |
| 192.168.161.17 | k8s-product-node01 | master |
| 192.168.161.18 | k8s-product-node02 | master |
| 192.168.161.19 | k8s-product-node03 | master |
| 192.168.161.20 | k8s-product-node04 | worker |
| 192.168.161.21 | k8s-product-node05 | worker |
| 192.168.161.22 | k8s-product-node06 | worker |
| 192.168.161.23 | k8s-product-node07 | worker |

## 关联的mysql服务信息


| ip            | 端口 | 角色 |
| ------------- | ---- | ---- |
| 192.168.161.3 | 3306 | 主   |
| 192.168.161.4 | 3306 | 从   |
| 192.168.161.5 | 3306 | 从   |



## 关联的harbor信息

| ip              | 端口  | 账号  |
| --------------- | ----- | ----- |
| 192.168.160.125 | 11180 | admin |



# 基础环境搭建

## 配置host

```sh
sudo vim /etc/hosts
```

```
192.168.161.17    k8s-product-node01
192.168.161.18    k8s-product-node02
192.168.161.19    k8s-product-node03
192.168.161.20    k8s-product-node04
192.168.161.21    k8s-product-node05
192.168.161.22    k8s-product-node06
192.168.161.23    k8s-product-node07
```

## 暂时关闭防火墙,服务搭建好后开启

```shell
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sudo setenforce 0
```

​	检查防火墙状态

```shell
sudo systemctl status firewalld.service
sudo iptables -L
sudo getenforce
```

## 设置 系统Limits限制

```shell
#临时切到root权限，否则前三条权限不够
sudo su -

echo -e "*\thard\tnofile\t1048576" >> /etc/security/limits.conf
echo -e "*\tsoft\tnofile\t655350" >> /etc/security/limits.conf
echo -e "*\tsoft\tcore\t10000000" >> /etc/security/limits.conf
/bin/sed -i 's/nproc     4096/nproc     65535/g' /etc/security/limits.d/20-nproc.conf
/bin/sed -i 's/#DefaultLimitNOFILE=/DefaultLimitNOFILE=655350/g' /etc/systemd/system.conf
/bin/sed -i 's/#DefaultLimitNPROC=/DefaultLimitNPROC=65535/g' /etc/systemd/system.conf

#退出root用户
exit  

```

## 禁用SWAP

```shell
#临时切到root权限
sudo su -

swapoff -a
/bin/sed -i "s/.*swap.*/#&/" /etc/fstab
/bin/sed -i "s/.*swappiness.*/#&/" /etc/sysctl.conf
/bin/echo -e "vm.swappiness = 0" >> /etc/sysctl.conf
sysctl -p

#退出root用户
exit  
```

## 开启IP转发

```shell
sudo su -   #临时切到root权限)
#必须顶格写
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl -p
exit  #退出root用户
```

## 安装docker

- 上传安装包docker1903.zip到指定的安装目录
- 解压安装包: unzip docker1903.zip
- 进入解压后的目录 : cd docker1903
- 执行安装命令: sudo rpm -ivh *.rpm --force --nodeps

## 配置docker registry

- 临时切到root权限: sudo su - 

- 创建配置文件所在的目录: sudo mkdir -p /etc/docker/

- sudo touch /etc/docker/daemon.json

  ```shell
  cat > /etc/docker/daemon.json<<EOF
  {
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
      "max-size": "10m",
      "max-file": "3"
      },
  "insecure-registries": ["127.0.0.1","192.168.160.125:11180"],
  "data-root":"/data/var/lib/docker"
  }
  EOF
  ```

- 添加docker用户组,命令如下:

  ```shell
  groupadd docker
  usermod -aG docker app
  systemctl restart docker
  ```

- 退出root用户: exit

- 验证

```shell
sudo docker login 192.168.160.125:11180
输入harbor对应的账号信息
```

## 配置jdk环境

- 上传安装包到指定目录

- 解压安装包并重命名,最终安装目录为/data/java

- 编辑/etc/profile文件,添加如下信息

  ```shell
  export JAVA_HOME=/data/java
  export JRE_HOME=${JAVA_HOME}/jre
  export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
  export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
  export PATH=$PATH:${JAVA_PATH}
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

## 配置免秘钥登陆

各机器需要配置root免秘钥登陆,需要用到root权限,具体过程此处不再赘述



# 集群安装

## k8s安装步骤

- 上传安装 k8s-install.zip,并解压到/data目录下

- cd /data/k8s-install

- 执行如下命令:

  ```shell
  sh installk8s.sh init --master 192.168.161.17 \
      --master 192.168.161.18 \
      --master 192.168.161.19 \
      --node 192.168.161.20 \
      --node 192.168.161.21 \
      --node 192.168.161.22 \
      --node 192.168.161.23 \
      --version v1.19.0 \
      --pkg-url /data/k8s-install/kube1.19.0.tar.gz
  ```

  

- 等待大概5分钟

- 输出"successd"，代表安装成功

- 验证
  kubectl get cs,输出以下信息为正常

  ```
  NAME                 STATUS    MESSAGE             ERROR
  etcd-0               Healthy   {"health":"true"}   
  scheduler            Healthy   ok                  
  controller-manager   Healthy   ok    
  ```

  

## 防火墙设置

防火墙相关命令:

```shell
systemctl start firewalld
systemctl stop firewalld
systemctl status firewalld
systemctl enable firewalld

#修改防火墙规则后重新加载配置
firewall-cmd --reload

#查看防火墙端口信息
firewall-cmd --list-port

#查看防火墙配置的所有规则
firewall-cmd --list-all
```



在防火墙开启的状态下执行如下命令:

```
firewall-cmd --zone=public --add-port=30080/tcp --permanent
firewall-cmd --zone=public --add-port=30070/tcp --permanent
firewall-cmd --zone=public --add-port=10248/tcp --permanent
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10255/tcp --permanent
firewall-cmd --zone=public --add-port=45538/tcp --permanent
firewall-cmd --zone=public --add-port=10249/tcp --permanent
firewall-cmd --zone=public --add-port=10256/tcp --permanent
firewall-cmd --zone=public --add-port=8443/tcp --permanent
firewall-cmd --zone=public --add-port=9099/tcp --permanent
firewall-cmd --zone=public --add-port=179/tcp --permanent
firewall-cmd --zone=public --add-port=2379/tcp --permanent
firewall-cmd --zone=public --add-port=2380/tcp --permanent
firewall-cmd --zone=public --add-port=10251/tcp --permanent
firewall-cmd --zone=public --add-port=10259/tcp --permanent
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --zone=public --add-port=9100/tcp --permanent
firewall-cmd --zone=public --add-port=3000/tcp --permanent
firewall-cmd --zone=public --add-port=9090/tcp --permanent
firewall-cmd --zone=public --add-port=10252/tcp --permanent
firewall-cmd --zone=public --add-port=10257/tcp --permanent
firewall-cmd --zone=public --add-port=111/tcp --permanent
firewall-cmd --zone=public --add-port=8070/tcp --permanent
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-port=8090/tcp --permanent
firewall-cmd --zone=public --add-port=9001/tcp --permanent
firewall-cmd --zone=public --add-port=9002/tcp --permanent
firewall-cmd --zone=public --add-port=53/udp --permanent
firewall-cmd --zone=public --add-port=19001-19500/tcp --permanent

firewall-cmd --reload
```


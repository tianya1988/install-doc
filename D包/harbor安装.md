# 安装harbor

#基础环境参考基础环境文档。

```shell

#安装harbor需要先安装docker,docker安装参考docker安装文档。


#上传harbor离线安装包至/data/install-tools/harbor
cd /data/install-tools/
tar xf harbor-offline-installer-v1.8.3.tgz -C /opt/
cd /opt
mv harbor/ harbor-v1.8.3
ln -s /opt/harbor-v1.8.3/ /opt/harbor

#创建数据目录和日志目录
mkdir -p /data/harbor /data/harbor/logs


#修改/opt/harbor/harbor.yml文件以下内容
hostname: 192.168.160.125
http:
  port: 11180
data_volume: /data/harbor
location: /data/harbor/logs

#安装docker-compose 
cp /data/install-tools/harbor/docker-compose /usr/local/bin/
chmod +x /usr/local/bin/docker-compose

#安装harbor
sh /opt/harbor/install.sh

 

#停止harbor
cd /opt/harbor && docker-compose down

#启动harbor
cd /opt/harbor && docker-compose up -d


#harbor登录地址：HarborIP:11180
#默认登录用户名密码 admin/******    #密码可在harbor.yml中设置
```


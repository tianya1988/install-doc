docker 安装手册



#基础环境详见基础环境文档。

# 安装docker



```shell
#将docker1809离线包上传到服务器任意目录下/data/install_tools/docker1903

cd /data/install-tools/docker1903/
rpm -ivh *.rpm --force --nodeps
```

 

# 将普通用户加入docker组



```shell
groupadd docker
usermod -aG docker ${USER}
```



# 配置docker registry

 

```shell
vim  /etc/docker/daemon.json

{
 "max-concurrent-downloads": 10,
 "log-driver": "json-file",
 "log-level": "warn",
 "log-opts": {
  "max-size": "10m",
  "max-file": "3"
  },
 "insecure-registries":
    ["127.0.0.1","harborIP:11180"],
 "data-root":"/var/lib/docker"
}

#如果允许连外网,需要从外网下载镜像,比如开发环境,可以添加如下配置:
"registry-mirrors": [
   "https://bxsfpjcb.mirror.aliyuncs.com"
 ],

#修改完成后,重启daemon-reload,然后重启docker
systemctl daemon-reload
systemctl restart docker

#把docker加入开机自启
systemctl enable docker
```



 


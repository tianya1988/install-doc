
# OpenResty安装手册

#基础环境详见基础环境文档。

## 服务器信息 

| ip            | 服务      | 端口 |
| ------------- | --------- | ---- |
| 192.168.161.1 | OpenResty | 8080 |
| 192.168.161.2 | OpenResty | 8080 |

## 上传安装包

上传安装包到/data/install_packge目录

## 解压

进入/data/install_packge

```
cd /data/install_packge
tar -zxvf openresty-1.19.3.1.tar.gz
```

## 安装依赖

```shell
sudo yum install readline-devel pcre-devel openssl-devel -y
```

## 配置OpenResty

```
#进入openresty 解压目录
cd /data/install_packge/openresty-1.19.3.1
#配置
./configure --prefix=/data/openresty
```

## 编译安装

```
make && make install
```

## 目录介绍

```
#证书文件存放目录
/data/openresty/nginx/cert
#前端代码存放目录
/data/openresty/nginx/html
#nginx配置文件目录
/data/openresty/nginx/conf
#nginx日志文件目录
/data/openresty/nginx/logs
#nginx启停脚本目录
/data/openresty/nginx/sbin
```

## 命令

```
#启动
/data/openresty/nginx/sbin/nginx
#停止
/data/openresty/nginx/sbin/nginx -s stop
#重启
/data/openresty/nginx/sbin/nginx -s reload
```


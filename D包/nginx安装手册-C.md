nginx安装手册


#基础环境详见基础环境文档。

# nginx安装手册

## 服务器信息

| ip             | 端口 |
| -------------- | ---- |
| 192.168.161.45 | 8080 |
| 192.168.161.46 | 8080 |

## 上传安装包到/data/install-packages

## 安装依赖

```bash
sodo yum install gcc-c++
sodo yum install pcre*
sodo yum install openssl*
```

## 解压

```bash
tar -zxvf nginx-1.19.4.tar.gz
```

## 进入文件夹nginx-1.19.4

```bash
cd nginx-1.19.4
```

## 配置nginx

```bash
./configure \
	--user=app \
	--group=app \
	--prefix=/data/nginx \
	--with-http_ssl_module \
	--with-http_stub_status_module \
	--with-http_realip_module \
	--with-threads
```

## 编译安装

```bash
make && make install
```

## 配置系统服务

```yaml
sudo vim /usr/lib/systemd/system/nginx.service
#写入以下内容

# /usr/lib/systemd/system/nginx.service
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/data/nginx/logs/nginx.pid
ExecStartPre=/data/nginx/sbin/nginx -t -c /data/nginx/conf/nginx.conf
ExecStart=/data/nginx/sbin/nginx -c /data/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## 重新加载nginx服务程序的配置文件

```shell
sudo systemctl daemon-reload
```

## nginx命令

```shell
#启动
sudo systemctl start nginx
#停止
sudo systemctl stop nginx
#重启
sudo systemctl restart nginx
```


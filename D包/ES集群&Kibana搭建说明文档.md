# ES集群&Kibana搭建说明文档

#基础环境见基础环境文档。

## 1. 授权用户目录操作权限

``` shell
# 切换到root用户
sudo su -
# 赋予app用户文件夹权限 
chown -R app:app /data
# 推出root
exit
```

## 2. 上传压缩包并解压

``` shell
# 进入data目录
cd /data
# 选择文件上传
sudo rz
# 解压elasticsearch压缩包
tar -zxvf elasticsearch-7.10.0-linux-x86_64.tar.gz
# 重命名文件夹
mv elasticsearch-7.10.0-linux-x86_64 elasticsearch
```

## 3.分别修改es集群节点配置文件

``` shell
# 打开配置文件
vim /data/elasticsearch/config/elasticsearch.yml
```

修改文件内容：

``` shell
#指定es集群名
cluster.name: es-cluster
#指定当前es节点名
node.name: node-1
#非数据节点
node.data: false
#如果master节点设置为true
node.master: true
#自定义的属性,这是官方文档中自带的
node.attr.rack: r1
#开启启动es时锁定内存
bootstrap.memory_lock: true
#当前节点的ip地址
network.host: 172.17.0.5
#设置当前节点占用的端口号，默认9200
http.port: 9200
#启动当前es节点时会去这个ip列表中去发现其他节点，此处不需配置自己节点的ip,这里支持ip和ip:port形式,不加端口号默认使用ip:9300去发现节点
discovery.seed_hosts: ["172.17.0.3:9300","172.17.0.4:9300","172.17.0.2:9300"]
#可作为master节点初始的节点名称,tribe-node不在此列，只写作为master节点的名称
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
#设置集群中N个节点启动时进行数据恢复，默认为1。（可选项）
gateway.recover_after_nodes: 2
 #数据保存目录
path.data: /es/data
#日志保存目录
path.logs: /es/logs
#设置集群节点发现的端口
transport.tcp.port: 9300
# 用户权限配置，可选，为了安全考虑推荐配置
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

## 4.生成集群间证书认证（可选，为了安全考虑推荐配置）

``` shell
# 进入elasticsearch目录
cd elasticsearch
# 执行生成证书
bin/elasticsearch-certutil ca
# 直接回车默认是生成文件 elastic-stack-ca.p12 ，后面就回车就行
Please enter the desired output file [elastic-stack-ca.p12]: 
... 回车 ... 生成elastic-stack-ca.p12文件
# 生成 elastic-stack-ca.p12后，执行命令elasticsearch-certutil
bin/elasticsearch-certutil cert --ca /elasticsearch/elastic-stack-ca.p12
... 回车 ... 生成elastic-certificates.p12文件
# 将生成的文件拷贝到各个节点的config目录下
```

## 5.修改系统用户限制

``` shell
# 切换到root用户
sudo su -
# 打开并编辑 /etc/security/limits.conf
vim /etc/security/limits.conf
# 在最后增加下面内容
app soft nofile 65536
app hard nofile 65536
app soft nproc 4096
app hard nproc 4096
app hard memlock unlimited
app soft memlock unlimited
# 更新配置文件
sysctl -p

# 打开并编辑 /etc/systemd/system.conf
vim /etc/systemd/system.conf
DefaultLimitNOFILE=65536
DefaultLimitNPROC=32000
DefaultLimitMEMLOCK=infinity
```

## 6.重启操作系统使配置文件生效

``` shell
sudo reboot -f
```

## 7.开启认证

**在开启认证之前，如果有多个master节点，只启动一个master节点进行认证设置，不然密码设置不成功**

``` shell
# 设置认证密码
/data/elasticsearch/bin/elasticsearch-setup-passwords interactive
# 等几秒后开始提示对应的信息，输入2次y，然后根据实际情况输入对应用户的密码
```

## 8.启动其他节点

``` shell
# 启动命令
/data/elasticsearch/bin/elasticsearch
# 后台启动命令
/data/elasticsearch/bin/elasticsearch -d
```

## 9.安装kibana

``` shell
# 进入data目录
cd /data
# 选择文件上传
sudo rz
# 解压kibana压缩包
tar -zxvf kibana-7.10.0-linux-x86_64.tar.gz
# 重命名文件夹
mv kibana-7.10.0-linux-x86_64 kibana
```

## 10.修改kibana配置文件

``` shell
# 打开并编辑配置文件
vim /data/kibana/config/kibana.yml
```

在配置文件中修改一下内容：

``` shell
# 端口
server.port: 5601
# IP
server.host: "192.168.161.94"
# es
elasticsearch.hosts: ["http://192.168.161.94:9200","http://192.168.161.95:9200","http://192.168.161.96:9400"]
# 国际化-中文设置
i18n.locale: "zh-CN"
# es登录认证配置
elasticsearch.username: "elastic"
elasticsearch.password: "*******"   #******代表要设置的密码
# 如果需要nginx转发访问，需要配置下面信息
server.basePath: "/elk" # 转发的url
server.rewriteBasePath: false
```

## 11.启动kibana

``` shell
# 前台启动
/data/kibana/bin/kibana
# 后台启动
nohup /data/kibana/bin/kibana > /dev/null 2>&1 & 
```




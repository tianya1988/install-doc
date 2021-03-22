
#基础环境详见基础环境文档。

# 安装步骤



```shell
#主192.168.161.89
#从192.168.161.90

(1)在线安装 
yum install bind bind-utils -y

(2)离线安装
将离线安装包文件夹bind上传到服务器/data/install-tools/bind
cd /data/install-tools/bind
rpm -ivh *.rpm --force --nodeps
```



# 修改配置文件/etc/named.conf

## 主服务器

(1)修改网络设置

```shell
listen-on port 53 { 192.168.161.89; };  #表示监听10.1.20.43的53端口
allow-query     { 192.168.161.0/24; };  #表示允许10.1.20.0/24网段的主机查询
```

(2)开启向其他DNS转发

```shell
     forward first;
     forwarders {
          192.168.2.123;
          192.168.2.122;
      };
     dnssec-enable no;
     dnssec-validation no;
```

(3)开启dns查询记录日志
在logging添加

         channel query_log {
                 file "data/query.log" versions 4 size 20m;
                 severity debug 3;
                 print-time yes;
                 print-category yes;
        };
        category security {
                 query_log;
        };

(4)添加域设置,如有多个域,就添加多个zone。

```
zone "cscec-sc.in" IN {
        type master;
        file "cscec-sc.in.zone";
        allow-update { 192.168.161.89; };
        allow-transfer { 192.168.161.90; };
};
```



## 从服务器 

(1)修改网络设置

```shell
listen-on port 53 { 192.168.161.90; };  #表示监听10.1.20.43的53端口
allow-query     { 192.168.161.0/24; };  #表示允许10.1.20.0/24网段的主机查询
```

(2)开启向其他DNS转发

         forward first;
         forwarders {
              192.168.2.123;
              192.168.2.122;
          };
         dnssec-enable no;
         dnssec-validation no;


(3)开启dns查询记录日志
在logging添加

         channel query_log {
                 file "data/query.log" versions 4 size 20m;
                 severity debug 3;
                 print-time yes;
                 print-category yes;
        };
        category security {
                 query_log;
        };

(4)添加域设置,如有多个域,就添加多个zone。

```
zone "cscec-sc.in" IN {
        type slave;
        file "cscec-sc.in.zone";
        masters { 192.168.161.89; };
};
```



# 创建DNS库文件

文件位于/var/named/，新添加几个域就创建几个库文件

编辑cscec-sc.in.zone

```
$ORIGIN cscec-sc.in.
$TTL 3H
@       IN SOA  @  mail.cscec-sc.in (
                                        0       ;serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        IN NS   @
        IN A    192.168.161.89
www     IN A    192.168.161.89
webgw-a-pro  IN A    192.168.161.99
webgw-c-pro  IN A    192.168.161.102
dbsrv-zh-pro IN A    192.168.161.98
```

给DNS库文件赋权限

```shell
chown named:named cscec-sc.in.zone
```



# 检查配置

```shell
named-checkconf  #检查/etc/named.conf语法,不报错即可
named-checkzone cscec-sc.in /var/named/cscec-sc.in.zone  #检查zjzb.com这个域的库文件,全部正确会显示OK
systemctl restart named #重启named服务
rndc reload #重新加载rndc服务

systemctl status named  #查看named服务状态
systemctl enable named  #设置named服务开机自启
```



# 添加防火墙规则

```shell
sudo systemctl start firewalld 
sudo firewall-cmd --zone=public --add-port=53/udp --permanent
sudo firewall-cmd --zone=public --add-port=53/tcp --permanent
sudo firewall-cmd --reload
```



# 将主机的dns改为192.168.161.89 和 192.168.161.90

```shell
sed -i "s/192.168.2.123/192.168.161.89/g" /etc/sysconfig/network-scripts/ifcfg-eth0  
sed -i "s/192.168.2.122/192.168.161.90/g" /etc/sysconfig/network-scripts/ifcfg-eth0 
systemctl restart network 

#然后 ping域名，查看解析出来的IP
ping webgw-a-pro.cscec-sc.in
ping webgw-c-pro.cscec-sc.in
ping dbsrv-zh-pro.cscec-sc.in
```



# 查看dns的查询记录

```shell
cat /var/named/data/query.log 
```







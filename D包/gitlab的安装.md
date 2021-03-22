# gitlab的安装

#基础环境详见基础环境文档。

## 在线安装



```shell
#配置yum源
cat << eof > /etc/yum.repos.d/gitlab-ce.repo
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
gpgcheck=0
enabled=1
eof

#更新本地yum缓存
yum clean all
yum makecache

#安装GitLab社区版
yum install gitlab-ce -y #自动安装最新版
yum install gitlab-ce-x.x.x #安装指定版本
```



## 离线安装



```shell
#将离线安装包上传到/data/install-tools/gitlab

cd /data/install-tools/gitlab
rpm -ivh *.rpm --force --nodeps
```



# gitLab常用命令



```shell
gitlab-ctl start      # 启动所有 gitlab 组件
gitlab-ctl stop       # 停止所有 gitlab 组件
gitlab-ctl restart    # 重启所有 gitlab 组件
gitlab-ctl status     # 查看服务状态
gitlab-ctl reconfigure # 重新生成配置
/etc/gitlab/gitlab.rb  # 默认的配置文件位置
gitlab-rake gitlab:check SANITIZE=true --trace  # 检查gitlab
gitlab-ctl tail # 查看日志
```


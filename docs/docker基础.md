### 1.安装docker

```shell
yum -y install yum-utils

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

--禁用docker测试版本--

yum-config-manager --disable docker-ce-edge
yum-config-manager --disable docker-ce-test  
yum list docker-ce --showduplicates|sort -r

--卸载旧版本--
yum remove docker-ce docker-ce-cli

--安装指定版本--

yum -y install docker-ce-18.09.0-3.el7
```

### 2.获取gitlab镜像

```shell
docker pull gitlab/gitlab-ce:12.10.6-ce.0
```

### 3.启动容器

```shell
docker run \
    -d \
    -p 8443:443 \
    -p 8090:80 \
    --name gitlab \
    --restart unless-stopped \
    -v /opt/gitlab/etc:/etc/gitlab \
    -v /opt/gitlab/log:/var/log/gitlab \
    -v /opt/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:12.10.6-ce.0
```

说明：

--detach 后台启动
-p 容器的端口映射
--name 容器的名字
--restart always 当容器退出或宿主机重启的时候，容器接着会始终重启
-v 给容器添加一个数据卷

主机目录提前创建完毕
-v /opt/gitlab/etc:/etc/gitlab \
-v /opt/gitlab/log:/var/log/gitlab \
-v /opt/gitlab/data:/var/opt/gitlab \



### 4.修改external_url

```
vim /opt/gitlab/etc/gitlab.rb
修改 external_url 'http://gitlab.bidduo.com.cn'
```




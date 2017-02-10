## harbor 介绍
harbor是私有的docker镜像仓库，扮演着非常重要的角色。
```
 +---------+     +--------+         +----------+                            
 | git     |---->|   CI   |-------->| harbor   |                           
 +---------+     +--------+         +----------+                           
     ^                                   |                         +----------------+
     |                       +-----------|----------+              |     shipyard   |    
  developer                  |           |          |              +----------------+ 
                             V           V          V                       |         
                         +------+   +--------+  +-------+          +-----------------+
                         | node |   | node   |  | node  |  <====>  | swarm  manager  |
                         +------+   +--------+  +-------+          +-----------------+
                             |          |          |                        |        
                             |      +------------------+                    |       
                             |______|  etcd cluster    |____________________|      
                                    +------------------+ 
```
图中可以看到，CI系统会将镜像push给harbor，swarm集群要运行一个容器时，swarm节点会去仓库中拉取镜像。

## 设置主机名
```
[root@yjybj-3-031 iat]# cat /etc/hosts
127.0.0.1   localhost
172.28.4.11 yjybj-4-011 reg.iflytek.com
```
每个需要拉取镜像的节点都需要解析reg.iflytek.com仓库主机名地址

## 安装harbor
1. 下载一个[离线包](https://github.com/vmware/harbor/releases/download/0.5.0-rc1/harbor-offline-installer-0.5.0-rc1.tgz)
2. 解压后修改harbor.cfg里的hostname=reg.iflytek.com  里面也可配置admin的密码莫认：Harbor12345
3. 执行install.sh
4. 浏览器访问80端口

## 设置docker engine参数
vim /usr/lib/systemd/system/docker.service

ExecStart= 增加参数：
 `--insecure-registry reg.iflytek.com`

重启docker engine:
```
$ systemctl daemon-reload
$ service docker restart
```

## 创建项目
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/harbor-project.png)

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/harbor-open.png)是否公开点成【是】，否则swarm节点没有权限拉取会导致错误

登陆仓库(输入用户名密码)：
```
$ docker login reg.iflytek.com
```

将本地一个ubuntu:12.04镜像上传到harbor的develop项目下：
```
$ docker tag ubuntu:12.04 reg.iflytekc.com/develop/ubuntu:12.04
$ docker push reg.iflytekc.com/develop/ubuntu:12.04
```

## 设置同步

增加策略：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/admin-sync.png)
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/add-sync.png)

设置使用上面配置好的策略:

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/project-sync.png)

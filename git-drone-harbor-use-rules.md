## 持续集成环境使用准则
```
     +-----------------------------administrator---------------------------+
     |               |                  |                                  |
     V               V                  V                                  |
 +---------+     +--------+         +----------+                           | 
 | git     |---->| drone  |-------->| harbor   |                           |
 +---------+     +--------+         +----------+                           V
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
流程：

* 开发者将代码push到git仓库，git上设置了一些钩子，通知持续继承组建drone
* drone 收到git事件自动将代码拉下来开始构建，构建完成打包成docker镜像并推送到harbor镜像仓库
* 管理员通过shipyard(或者docker-compose)执行一个创建容器的指令
* swarm manager收到创建容器的指令，选择执行改容器合适的节点，并将执行指令发送给对应节点的docker engine
* 特定节点的docker engine发现节点上没有对应容器的镜像，自动去harbor镜像仓库拉取，并启动容器

## git仓库使用
本着简单并满足以下需求的原则：

需求：

* 每次使用git打tag时触发自动构建docker镜像过程 
* 开发人员没有修改主仓库的权限，只有通过pull request的方式然后主仓库管理员merge到主仓库中

```
    admin                    developer
  +-------+     fork        +-------+
  | repo  |---------------->| repo  |
  +-------+                 +-------+
      |                         |
      |                         o tag alpha-0.1.1
      |                         |
      |                         o tag alpha-v0.1.2
      |                         |
      |      pull request       o tag alpha-v1.0
merge o<------------------------|
      |                         |
      o tag beta-v1.0           o tag alpha-v1.1.1
      |                         |
      o tag release-v1.0        |
      |                         |
      V                         V
```
> admin创建项目
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/add-repo-0.png)
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/add-repo-1.png)
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/add-repo-2.png)
> developer fork该项目
开发者登陆开发都是gogs账号，从admin仓库里fork一份出来进行开发：

> developer tag触发构建alpha版本镜像
> developer pull request
> 测试人员tag触发构建beta版本镜像
> 测试人员tag触发构建release版本镜像

## 开发环境->测试/准上线环境->线上环境

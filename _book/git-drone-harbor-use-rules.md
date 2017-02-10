## 持续集成环境使用准则
```
     +-----------------------------administrator---------------------------+
     |               |                  |                                  |
     V               V                  V                                  |
 +---------+     +--------+         +----------+                           | 
 | git     |---->| drone  |-------->| harbor   |                           |
 +---------+     +--------+         +----------+                           V
     ^                                   |                         +----------------+
     |                       +-----------|----------+              |     dface      |    
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

## git push 触发构建

drone 默认每次push会触发构建docker镜像，要想push的时候不去构建镜像，在commit的rmessage里加入 “[CI SKIP]”即可，如下：

```
fanuxdeMacBook-Air:hello-world fanux$ git commit -m "[CI SKIP] not CI"
[master 0e3fa3f] [CI SKIP] not CI
 1 file changed, 1 insertion(+)
fanuxdeMacBook-Air:hello-world fanux$ git push
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 353 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
To http://localhost:3000/fanux/hello-world.git
   f279f0d..0e3fa3f  master -> master
```

## git仓库使用
本着简单并满足以下需求的原则：

需求：

* 每次使用git push操作时触发自动构建docker镜像过程 
* 如只push不想构建 如上，commit 时加`[CI SKIP]`参数
* 开发人员没有修改主仓库的权限，只有通过pull request的方式然后主仓库管理员merge到主仓库中
* 测试人员不操作代码仓库,只操作镜像仓库

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

开发者登陆开发者的gogs账号，从admin仓库里fork一份出来进行开发：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/developer-login.png)

点击派生按钮：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/fork.png)
![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/fork-1.png)

> developer tag触发构建alpha版本镜像

开发者clone派生出来的项目：
```
fanuxdeMacBook-Air:repo-de fanux$ git clone http://192.168.86.92:3000/fanux/repo
Cloning into 'repo'...
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
Checking connectivity... done.
```

添加代码，打tag并提交：
```
fanuxdeMacBook-Air:repo fanux$ touch Dockerfile
fanuxdeMacBook-Air:repo fanux$ echo "FROM centos:7" >>Dockerfile
fanuxdeMacBook-Air:repo fanux$ git add .
fanuxdeMacBook-Air:repo fanux$ git commit -m "add docker file"
[master 03fd680] add docker file
 1 file changed, 1 insertion(+)
 create mode 100644 Dockerfile
fanuxdeMacBook-Air:repo fanux$ git tag alpha-v0.1.0
fanuxdeMacBook-Air:repo fanux$ git push
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 287 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To http://192.168.86.92:3000/fanux/repo
   aa7bc47..03fd680  master -> master
fanuxdeMacBook-Air:repo fanux$ git push origin --tags        # 这个动作会触发drone自动构建docker镜像并提交到镜像仓库
Total 0 (delta 0), reused 0 (delta 0)
To http://192.168.86.92:3000/fanux/repo
 * [new tag]         alpha-v0.1.0 -> alpha-v0.1.0
```
如此就能看到tag了：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/show-tags.png)

> developer pull request

开发者点击绿色按钮，创建pull request,请求合并 

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/pull-request-0.png)

点击创建合并请求:

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/pull-request-1.png)

管理员主仓库就能看到合并请求了：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/merge-1.png)

管理员点击合并请求：

![](http://192.168.86.170:10080/iflytek/docs/raw/master/images/merge-2.png)

## TODOharbor仓库使用

## 开发环境->测试->准上线环境/线上环境
* 仅开发环境有持续集成的组建
* 测试和线上环境只有镜像仓库和swarm等组建
* 三个环境仓库主机名相同，都为reg.iflytek.com

方案1：
```
  +-------+       +--------------+      
  | drone |------>| dev registry |----+      
  +-------+       +--------------+    | docker pull reg.iflytek.com/dev/hello-world:alpha-v1.0  
                                      | docker tag reg.iflytek.com/dev/hello-world:alpha-v1.0  reg.iflytek.com/test/hello-world:alpha-v1.0  
                                      | docker save reg.iflytek.com/test/hello-world:alpha-v1.0 > hello-world.tar
                                      | scp ...
                                      | docker load -i hello-world.tar
                                      | docker push reg.iflytek.com/test/hello-world:alpha-v1.0  
                  +---------------+ <-+ 
                  | test registry | 
                  +---------------+ --+   
                                      | docker pull reg.iflytek.com/test/hello-world:alpha-v1.0  
                                      | docker tag reg.iflytek.com/test/hello-world:alpha-v1.0  reg.iflytek.com/release/hello-world:alpha-v1.0  
                                      | docker save reg.iflytek.com/release/hello-world:alpha-v1.0 > hello-world.tar
                                      | scp ...
                                      | docker load -i hello-world.tar
                                      | docker push reg.iflytek.com/release/hello-world:alpha-v1.0  
               +------------------+   |
               | release registry |<--+
               +------------------+
```

方案2（镜像仓库同步）:
* 环境网络相通
```
             +-------+       +--------------+
             | drone |------>| dev registry | dev.reg.iflytek.com ----- dev
             +-------+       +--------------+                       |__ test
                                    | sync dev project
                                    V
                             +---------------+ ---+ docker pull dev.reg.iflytek.com/dev/hello-world:alpha-v1.0
      test.reg.iflytek.com   | test registry |    | docker tag dev.reg.iflytek.com/dev/hello-world:alpha-v1.0 test.teg.iflytek.com/test/hello-world:alpha-v1.0
              |____dev       +---------------+ <--+ docker push test.teg.iflytek.com/test/hello-world:alpha-v1.0
              |____test             | sync  test project
              |____rel              V
                             +------------------+ ---+ docker pull test.teg.iflytek.com/test/hello-world:alpha-v1.0
       rel.reg.iflytek.com   | release registry |    | docker tag test.teg.iflytek.com/test/hello-world:alpha-v1.0 rel.teg.iflytek.com/rel/hello-world:alpha-v1.0
              |____test      +------------------+ <--+ docker push rel.teg.iflytek.com/rel/hello-world:alpha-v1.0
              |____rel
```

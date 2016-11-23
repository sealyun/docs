## 原理解析
考虑一下在多个节点上使用docker的场景：

没有swarm时(需要到每个节点上去运行docker)：
```
  docker   docker    docker
  run      run       run
   |        |         |
   V        V         V
  +------+ +------+ +------+ 
  | node | | node | | node |
  +------+ +------+ +------+
```
有了swarm(讲请求转发给对应主机上的docker engine):
```
  docker  +-------+
  run---->| swarm |
          +-------+
              |
     +--------+--------+
     |        |        |
     V        V        V
  +------+ +------+ +------+ 
  | node | | node | | node |
  +------+ +------+ +------+
```

swarm分两个组件，manager和join，swarm join逻辑非常简单，干的事也非常单一，就是把自己节点的engine地址注册到服务发现中，并维持一个心跳。
所有容器的操作都是由swarm manager直接发给对应节点上的docker engine的。
```
         +------+   +--------+  +-------+          +-----------------+ 
         | node |   | node   |  | node  |  <====>  | swarm  manager  |
         +------+   +--------+  +-------+          +-----------------+ 
            |          |          |                        |          
            |      +------------------+                    |         
            |______|  etcd cluster    |____________________|        
                   +------------------+                 
```
## swarm 安装
## 使用教程
swarm兼容了docker api, 意味着怎么使用docker就怎么使用swarm, 没有额外的学习成本。 假设swarm manager地址为 tcp://localhost:4000
```
$ docker run -H tcp://localhost:4000 (一切docker 命令)
```

### 过滤器
```
                           +---> Constraint:
                           |          给docker engine打标签：docker deamon --label storage=ssd
                           |          启动容器并调度到ssd节点上  docker -H tcp://swarm.iflytek.com:4000 -e constraint:storage==ssd nginx
        +---node filter----+
        |                  |                                                                 
        |                  |                                                                 
        |                  +---> containerslots:
        |                              节点上最多运行三个容器：docker daemon --label containerslots=3                              
        |                              部署nginx时每个节点最多部署两个: docker -H tcp://swarm.iflytek.com:4000 run -e containerslots=2 nginx                            
        |                                                                                    
filter--
        |                                                                                    
        |                      +----> affinity:                                                             
        |                      |           docker -H tcp://swarm.iflytek.com:4000 run -l foo=bar --name mysql mysql:latest 
        |                      |        麻烦调度nginx容器到一个运行有名字叫mysql容器的节点上：
        |                      |           docker -H tcp://swarm.iflytek.com:4000 run -e affinity:container==mysql nginx:latest 
        |                      |        请调度nginx到一个贴有foo=bar标签的容器运行的节点上去
        |                      |           docker -H tcp://swarm.iflytek.com:4000 run -e affinity:foo==bar nginx:latest 
        |                      |        请寻找到一个节点有nginx:latest镜像的节点上
        |                      |           docker -H tcp://swarm.iflytek.com:4000 run -e affinity:image==nginx:latest nginx:latest 
        |                      |        找一个有nginx:latest镜像的节点运行，如果找不到就随便找个节点运行（==~）约等于
        |                      |           docker -H tcp://swarm.iflytek.com:4000 run -e affinity:image==~nginx:latest nginx:latest 
        +---container filter---+                                                                       
                               |---> port 淘汰被占用端口号的节点
                               |
                               +---> dependency  依赖某些容器，卷，或者网络
```

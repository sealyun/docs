## why compose
通常docker命令很长，每次执行都去终端输入非常之麻烦。

有人觉得那也没必要用compose，写个shell脚本就行了。

可是有时不仅需要run，还需要stop，rm容器等等，以及定义容器之间的关系，启动多个容器，这些复杂的操作使用shell就比较麻烦了，compose解决了这些问题，通过定义一个yaml配置文件，实现对容器的操作（特别是创建容器的操作）

## compose 常用参数
创建一个目录，进入目录创建一个docker-compose.yml文件
compose里可定义多个容器，如把nginx mysql php三个容器定义在一个compose文件里。

compose有dependon参数定义容器的依赖关系，如某个容器依赖数据库，但不推荐使用dependon，因为容器启动成功了不代表里面的进程启动成功了。
事例：
```
version: '2'
services:
    service_iat:
        container_name: iat
        image: reg.iflytek.com/release/iat:0.9.4
        labels:
           - key=value
        devices:
           - "/dev/nvidia0:/dev/nvidia0"
        privileged: true
        environment:
            - "constraint:hostname==yjybj-2-010"
        command: sh run_iatserver.sh
        volumes:
           - /data/iat/resource:/root/resource
           - /data/iat/data:/root/data
           - /data/iat/conf/zkr.cfg:/root/sivs_run/bin/zkr.cfg
        network_mode: "host"
        restart: "on-failure:10"
```
配置文件可读性非常强，基本一眼就知道每个字段含义了

执行`docker-compose -f docker-compose.yml up -d` 等同于：
```
$ docker run -d --name iat \
             -l key=value \
             --device /dev/nvidia0:/dev/nvidia0 \
             --privileged  \
             -e constraint:hostname==yjybj-2-010 \
             -v /data/iat/resource:/root/resource \
             -v /data/iat/data:/root/data \
             -v /data/iat/conf/zkr.cfg:/root/sivs_run/bin/zkr.cfg \
             --net=host reg.iflytek.com/release/iat:0.9.4 sh run_iatserver.sh
```

`docker-compose stop|restart|down` 停止重启删除容器
* -f 指定配置文件，莫认使用当前目录下的docker-compose.yml文件

## 远程执行
有时compose文件在我们的pc上，但是我们想在服务器上执行一个容器. 假设远程服务器ip端口为 192.168.86.92:4000
```
docker-compose -H tcp://192.168.86.92:4000 up -d
```

## 批量部署
有时我们希望运行多个容器，这样可以使用compose的scale命令：
```
$ docker-compose scale service_iat=9   # 运行9个iat
```
这里需要注意，使用scale命令时compose文件里不能指定container-name, 否则会冲突，因为容器名唯一。  不指定时compose会自动命名，使用目录名+服务名+数字的方式给容器命名。

## 与swarm过滤器结合部署容器到指定节点
现需求如下：需要在主机名为yjybj-5-011 到 yjybj-5-019 的9个节点上部署nodemanage, 每个节点部署一个
```
...
        hadoop_node_manager_5_1x:
                network_mode: "host"
                environment:
                  - "constraint:hostname==yjybj-5-01[1-9]"
                  - "affinity:app!=nodemanager"
                labels:
                  - "app=nodemanager"
...
```
然后执行：`docker-compose -H tcp://swarm.iflytek.com:4000 scale hadoop_node_manager_5_1x=9` 即可
可以看到  我们使用`constraint:hostname==yjybj-5-01[1-9]` 来正则匹配节点，这样容器就不会跑到别的节点上运行，但是只有这个还不够，因为不能保证每个节点有且只有一个nodemanager容器。

所以我们使用非亲和性来保障这一点，先给容器贴标签：`app=nodemanager` 然后告诉集群不要把我运行在有`app=nodemanager`这个标签的容器的节点上：`affinity:app!=nodemanager`

```
version: '2'
services:
    sis-auth:
      command: sh watchdog.sh 172.27 172.27.0.13:2379/v2/keys 172.27.0.13:2379,172.27.3.30:2379,172.27.3.31:2379 online
      environment:
         - "constraint:hostname==yjybj-[04]-0(16|17|18|21|22|23)"
         - "affinity:app!=sis-auth"
         - TZ=Asia/Shanghai
      labels:
         - "app=sis-auth"
      network_mode: "host"
      volumes:
         - /data/sis/logs/sis-auth:/opt/server/logs
         - /etc/localtime:/etc/localtime
      image: reg.iflytek.com/release/sis-auth:0.9.1
```

更多资料请看官网：https://docs.docker.com/compose/overview/

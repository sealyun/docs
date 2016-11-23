## 基本概念
* 镜像(image) - 一堆文件和目录的集合，如centos镜像里面有/usr /lib /bin等目录。可类比成操作系统镜像。
* 容器(container) - 用过虚拟机的都知道如一个centos虚拟机镜像可以创建多个虚拟机，容器就可类比成虚拟机。很多资料上说容器是“运行着的镜像”其实不确切，容器产生误导，因为容器可以停止，停止了的容器就和关机的虚拟机一样也是存在的。

## 常用命令
![状态图](http://192.168.86.170:10080/iflytek/docs/raw/master/images/status.png)
```
$ docker run|create $(image)
$ docker ps
$ docker ps -a
$ docker start|stop|rm $(container)
$ docker rmi $(image)
$ docker tag $(image) $(new image)
$ docker pull|push $(image)
$ docker exec $(container) $(command)     如在容器中执行bash命令   docker exec -it $(container) /bin/bash
```
## 网络模式
## 磁盘挂载
## Dockerfile


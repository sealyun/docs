## drone:0.4安装
```
docker run \
    --volume /Users/fanux/CI/drone:/var/lib/drone \
    --volume /var/run/docker.sock:/var/run/docker.sock \
    -e REMOTE_DRIVER=gogs \
    #-e REMOTE_CONFIG=http://192.168.37.5:3000?open=false \
    -e REMOTE_CONFIG=http://192.168.37.5:3000?open=true \
    -e DATABASE_DRIVER=sqlite3 \
    -e DATABASE_CONFIG=/var/lib/drone/drone.sqlite \
    --publish=8000:8000 \
    --detach=true \
    --name=drone \
    drone/drone:0.4
```
在本机上创建一个目录，如：`/Users/fanux/CI/drone` 用于存储数据 

## 使用教程
浏览器访问8000端口，用gogs的账户和密码登陆即可。

然后就没有然后了。。。

## 配置

自己git项目的根目录下需要有个.drone.yml配置文件，一切都在配置文件中。如：

```
#build:
#   image: 192.168.86.106/develop/golang:latest
#   commands:
#      - go build

publish:
   docker:
      username: admin
      password: Harbor12345
      registry: 192.168.86.106
      email: fhtjob@hotmail.com
      repo: develop/hello-world
      tag: alpha-v1.0
      file: Dockerfile
      insecure: true
```

这样drone会根据项目根目录下的Dockerfile文件构建镜像，然后自动上传到镜像仓库。

## 构建与发布镜像分离
```
fanuxdeMacBook-Air:hello-world fanux$ cat .drone.yml
build:
   image: 192.168.86.106/develop/golang:latest
   commands:
      - go build -o hello-world

publish:
   docker:
      username: admin
      password: Harbor12345
      registry: 192.168.86.106
      email: fhtjob@hotmail.com
      repo: develop/hello-world
      tag: beta-v1.0
      file: Dockerfile
      insecure: true
fanuxdeMacBook-Air:hello-world fanux$ cat Dockerfile
FROM 192.168.86.106/develop/golang:latest
COPY hello-world .
CMD ./hello-world
```
这意味着可以把构建的结果分离到一个更小的镜像里去发布。

如构建时基础镜像需要jdk  maven等, 而发布仅需要一个jre的基础镜像。

## 容器中保留代码commit等信息
```
build:
   image: 192.168.86.106/develop/golang:latest
   commands:
      - go build -o hello-world
      - echo "LABEL commit=$$COMMIT branch=$$BRANCH build_number=$$BUILD_NUMBER" >> Dockerfile

publish:
   docker:
      username: admin
      password: Harbor12345
      registry: 192.168.86.106
      email: fhtjob@hotmail.com
      repo: develop/hello-world
      tag: release-v4.0
      file: Dockerfile
      insecure: true
```

## build 容器与工作容器的关系
看一个例子：
```
build:
   image: 192.168.86.106/devops/golang:1.7-godep
   commands:
     - mkdir -p $GOPATH/src/github.com/docker/swarm/ && cp -r ./* $GOPATH/src/github.com/docker/swarm/ && pwd
     - cd $GOPATH/src/github.com/docker/swarm/ && godep go build -o cattle && cd - && cp $GOPATH/src/github.com/docker/swarm/cattle .

publish:
   docker:
      username: admin
      password: Harbor12345
      registry: 192.168.86.106
      email: fhtjob@hotmail.com
      repo: devops/cattle
      tag: alpha-v1.0
      file: Dockerfile
      insecure: true
```

```
FROM 192.168.86.106/devops/alpine:3.4
COPY cattle /bin
CMD cattle --help
```
我们使用`golang:1.7-godep` 构建二进制程序，使用`alpine:3.4`发布二进制程序, 这样发布的镜像就可以非常小。

代码是在构建镜像外部被拉下来的，然后与构建镜像共享磁盘。  commands都是在容器内部执行的。

我们使用pwd可以看一下工作目录是什么：`/drone/src/192.168.37.5/fanux/cattle`， 这就是工作目录。所以我们在别的目录构建时需要把结果拷贝到这个文件。然后发布的Dockerfile才可以直接拷贝编译的结果。

## 安装
gogs安装：
```
docker run -d --name gogs-time -v /etc/localtime:/etc/localtime -e TZ=Asia/Shanghai --publish 8022:22 \
           --publish 3000:3000 --volume /data/gogs:/data gogs/gogs:0.10.18
```

drone安装，使用docker-compose:

如果没安装docker-compose
```
$ yum install python-setuptools
$ easy_install pip
$ pip install docker-compose
```

mkdir drone && cd drone && touch docker-compose.yml

cat docker-compose.yml :
```
version: '2'

services:
  drone-server:
    image: drone/drone:0.5
    ports:
      - 80:8000
    volumes:
      - ./drone:/var/lib/drone/
    environment:
      - DRONE_OPEN=true
      - DRONE_GOGS=true
      - DRONE_GOGS_URL=http://49.51.33.43:3000
      - DRONE_SECRET=fanux
#      - DRONE_GOGS_SKIP_VERIFY=true
#      - DRONE_GOGS_PRIVATE_MODE=true

  drone-agent:
    image: drone/drone:0.5
    command: agent
    depends_on: [ drone-server ]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_SERVER=ws://drone-server:8000/ws/broker
      - DRONE_SECRET=fanux
```
执行 `docker-compose up -d` 即可

## .drone.yml文件
```
pipeline:
    build:
       image: golang:1.8.0 #构建时用的临时容器
       commands:
         #此行必须加，使镜像关联代码的COMMIT ID等信息
         - echo "LABEL commit=$$COMMIT branch=$$BRANCH build_number=$$BUILD_NUMBER" >> Dockerfile
         - go build -o hello-drone

    publish:
          image: plugins/docker #插件镜像，无需修改
          registry: 10.1.86.51  #镜像仓库地址
          username: admin #镜像仓库用户名
          password: Harbor12345 #镜像仓库密码
          email: fhtjob@hotmail.com 
          repo: test/hello #项目/镜像名 
          #tag: latest #镜像tag
          tag: ${DRONE_TAG} #获取git tag的参数
          file: Dockerfile #交付时的Dockerfile,将构建完的目标文件（如bin文件，字节码）打包成新的镜像
          insecure: true
```

## Dockerfile
```
#交付时使用的基础镜像
FROM golang:1.8.0 
COPY hello-drone $GOPATH/bin 
CMD hello-drone 
```

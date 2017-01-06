## 反响代理，负载均衡器

### 启动traefik
docker-compose.yml:
```
traefik:
  image: traefik
  command: --web --docker --docker.domain=docker.localhost --logLevel=DEBUG
  ports:
    - "80:80"
    - "8080:8080"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /dev/null:/traefik.toml

whoami1:
  image: emilevauge/whoami
  labels:
    - "traefik.backend=whoami"
    - "traefik.frontend.rule=Host:whoami.docker.localhost"

whoami2:
  image: emilevauge/whoami
  labels:
    - "traefik.backend=whoami"
    - "traefik.frontend.rule=Host:whoami.docker.localhost"
```
这里启动了一个traefik和两个backend, backend是两个httpserver.
执行：`docker-compose up -d`

### 发送请求
docker ps 可看到：
```
CONTAINER ID        IMAGE               COMMAND                
74c7cd92fc1d        emilevauge/whoami   "/whoamI" 
dee2f0aa1f85        traefik             "/traefik --web --doc"  
8bace1428e88        emilevauge/whoami   "/whoamI"
```
```
╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 74c7cd92fc1d
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.4
IP: fe80::42:acff:fe11:4
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85
```

```
╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 8bace1428e88
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.2
IP: fe80::42:acff:fe11:2
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85
```
可以看到请求分别被两个容器处理。

### 停止一个backend
```
╰─➤  docker stop 8bace1428e88
```
```
╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 74c7cd92fc1d
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.4
IP: fe80::42:acff:fe11:4
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85

╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 74c7cd92fc1d
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.4
IP: fe80::42:acff:fe11:4
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85
```
可以看到两次请求都发送给了 `74c7cd92fc1d`, 容器完成了自动下线。

### 再次上线
```
docker start 8bace1428e88
```

```
─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 74c7cd92fc1d
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.4
IP: fe80::42:acff:fe11:4
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85

╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
Hostname: 8bace1428e88
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.2
IP: fe80::42:acff:fe11:2
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: curl/7.43.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Proto: http
X-Forwarded-Server: dee2f0aa1f85
```
可以看到`8bace1428e88` 完成了自动上线。

综上所述，taefik的一大优势是我们只需直接启动和关闭容器即可自动完成上下线。与nginx做反向代理的区别是我们需要
服务发现，然后根据服务发现修改nginx配置文件再reload.


问题：
1. 如何使用多个taefik实例。
2. 下线时正在处理的请求会不会失败。
3. 性能如何（与nginx对比）。


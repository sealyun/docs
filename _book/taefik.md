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

### 一个backend下线时，正在处理的请求会不会失败
实验思路：启动两个backend，traefik代理时会轮流发给两个backend，当客户端请求之后，关闭掉正在处理请求的backend容器。看结果

启动一个http服务，假设某个任务需要处理20秒
```
main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(time.Second * 20)
        fmt.Fprintf(w, "Hello, %q", os.Args)
    })

    log.Fatal(http.ListenAndServe(":80", nil))
}
```
打包成容器：
```
FROM golang:latest
COPY main.go .
LABEL traefik.backend=whoami
LABEL traefik.frontend.rule=Host:whoami.docker.localhost
CMD go run main.go
```
```
docker build -t backend:latest .
```
分别运行两个backend：
```
$ docker run -p 8082:80 --name=back1 backend:latest go run main.go back1
$ docker run -p 8082:80 --name=back2 backend:latest go run main.go back2
```
连续请求traefik:
```
curl -H Host:whoami.docker.localhost http://127.0.0.1
Hello, ["/tmp/go-build147686239/command-line-arguments/_obj/exe/main" "back2"]%   
curl -H Host:whoami.docker.localhost http://127.0.0.1
Hello, ["/tmp/go-build355301072/command-line-arguments/_obj/exe/main" "back1"]% 
curl -H Host:whoami.docker.localhost http://127.0.0.1
Hello, ["/tmp/go-build147686239/command-line-arguments/_obj/exe/main" "back2"]%
curl -H Host:whoami.docker.localhost http://127.0.0.1
Hello, ["/tmp/go-build355301072/command-line-arguments/_obj/exe/main" "back1"]%
```
可以看到两个backend轮流处理请求，实现负载均衡

现在向traefik请求一次，这次请求肯定是back2处理的，然后20秒之内关闭back2容器。
```
╰─➤  curl -H Host:whoami.docker.localhost http://127.0.0.1
```
关闭back2容器：
```
$ docker stop back2
```
返回bad gateway了：
```
Bad Gateway
```
结论：就目前看来，停止掉backend正在处理的请求会失败，traefik不会重新发送给别的backend处理

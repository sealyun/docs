## gogs
gogs是私有的git仓库，如使用github一样使用git

## 安装
```
docker run -d --name my-go-git-server --publish 8022:22 --publish 3000:3000 --volume /gogs-data/:/data gogs/gogs:latest
```
浏览器访问3000端口

然后，就没有然后了

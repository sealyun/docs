## 资源分发使用标准

### 文件/目录上传
#### 目录上传
iftp push [本地目录] [ftp服务器目录]
```
$ iftp push repo/ /user
```
执行完该命令，本地的repo/目录以及目录下的文件就会上传到ftp服务器的/user目录下。
push目录要加`/`

#### 文件上传
iftp push [当前目录下的文件] [ftp服务器目录]
```
$ iftp push README.md /user/repo
```
执行完该命令，本地的README.md文件就会上传到ftp服务器的/user/repo目录下。

### 文件/目录下载

#### 目录下载
iftp pull [ftp服务器目录] [本地目录,默认当前目录]
```
$ iftp pull /user/repo/ /root
```
将服务器repo目录下载到/root目录下

#### 文件下载
iftp pull [ftp服务器文件] [本地目录，默认当前目录]
```
$ iftp pull /user/repo/README.md /root
```
将README.md文件下载到/root目录下。

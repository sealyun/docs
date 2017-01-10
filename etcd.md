## etcd 增加节点
集群中本来有三个节点：
```
$ docker run --rm --net=host reg.iflytek.com/develop/etcd:2.3.1 etcdctl cluster-health
member 104a5791d6fd4860 is healthy: got healthy result from http://172.16.162.7:2379
member 185486cc02818a82 is healthy: got healthy result from http://172.16.162.6:2379
member a0e7ca9ea6dcc3a2 is healthy: got healthy result from http://172.16.162.4:2379
cluster is healthy
```

增加一个节点：http://172.16.162.10:2380
```
$ docker run --rm --net=host reg.iflytek.com/develop/etcd:2.3.1 etcdctl member add infra3 http://172.16.162.10:2380
Added member named infra3 with ID 18f7eaac4af0f5a3 to cluster

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra2=http://172.16.162.7:2380,infra1=http://172.16.162.6:2380,infra3=http://172.16.162.10:2380,infra0=http://172.16.162.4:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

到http://172.16.162.10:2380 上执行：
```
$ docker run -d --net=host \
   -e ETCD_NAME="infra3" \
   -e ETCD_INITIAL_CLUSTER="infra2=http://172.16.162.7:2380,infra1=http://172.16.162.6:2380,infra3=http://172.16.162.10:2380,infra0=http://172.16.162.4:2380" \
   -e ETCD_INITIAL_CLUSTER_STATE="existing" \
   -v /disk0:/etcd-data.etcd \
   reg.iflytek.com/develop/etcd:2.3.1 \
   etcd --listen-client-urls http://172.16.162.10:2379 \
        --advertise-client-urls http://172.16.162.10:2379 \
        --listen-peer-urls http://172.16.162.10:2380 \
        --initial-advertise-peer-urls http://172.16.162.10:2380 --data-dir /etcd-data.etcd
```

运行成功后即可看到：
```
docker run --rm --net=host reg.iflytek.com/develop/etcd:2.3.1 etcdctl cluster-health
member 104a5791d6fd4860 is healthy: got healthy result from http://172.16.162.7:2379
member 185486cc02818a82 is healthy: got healthy result from http://172.16.162.6:2379
member 18f7eaac4af0f5a3 is healthy: got healthy result from http://172.16.162.10:2379
member a0e7ca9ea6dcc3a2 is healthy: got healthy result from http://172.16.162.4:2379
cluster is healthy
```
最后，最好运行单数个节点

compose文件事例：
```
version: '2'
services:
    etcd_0_20:
       container_name: etcd_0_20
       network_mode: "host"
       environment:
           - ETCD_NAME=infra5 
           - ETCD_INITIAL_CLUSTER=infra2=http://172.27.3.31:2380,infra1=http://172.27.3.30:2380,infra3=http://172.27.0.14:2380,infra5=http://172.27.0.20:2380,infra0=http://172.27.0.13:2380,infra4=http://172.27.4.14:2380
           - ETCD_INITIAL_CLUSTER_STATE=existing 
           - "constraint:hostname==yjybj-0-020"
       volumes:
           - /data/etcd-data.etcd:/etcd-data.etcd
       image: reg.iflytek.com/release/etcd:2.3.1 
       command: |
             etcd --listen-client-urls http://172.27.0.20:2379 
                  --advertise-client-urls http://172.27.0.20:2379 
                  --listen-peer-urls http://172.27.0.20:2380 
                  --initial-advertise-peer-urls http://172.27.0.20:2380 --data-dir /etcd-data.etcd

```
```
$  docker-compose -H tcp://swarm.iflytek.com:4000 -f docker-compose-etcd-4-11.yml up -d
```

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

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

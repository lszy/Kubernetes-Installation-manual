# 目的 #
使用docker 部署etcd群集
# 需要主机 #
* etcd1.k8s.com
* etcd2.k8s.com
* etcd3.k8s.com
# 前提 #
已经安装docker，并正常运行
# 注意 #
本实验中，我们将在masters（etcds）.k8s.com上同时运行flannel和etcd，etcd要在flannel启动前进行相关部署，可以先启动etcd2，etcd3，然后启动etcd1的flanneld，然后启动etcd1的docker，启动etcd1的etcd1 容器部署。然后在在逐一修改etcd2和etcd3
# 群集部署 #
etcd1
```
docker run -tid -p 2379-2380:2379-2380 \
--name etcd1 \
-h etcd1 \
-e TZ=UTC-8 \
-v /opt/etcd:/var/lib/etcd/ \
--restart=always \
hub.k8s.com/google-containers/etcd:3.1.11 \
etcd -name etcd1 \
--data-dir=/var/lib/etcd/etcd1.etcd \
--heartbeat-interval='100' \
--election-timeout='1000' \
--listen-client-urls=http://0.0.0.0:2379 \
--listen-peer-urls http://0.0.0.0:2380 \
--advertise-client-urls=http://etcd1.k8s.com:2379 \
--initial-advertise-peer-urls=http://etcd1.k8s.com:2380 \
--initial-cluster etcd1=http://etcd1.k8s.com:2380,etcd2=http://etcd2.k8s.com:2380,etcd3=http://etcd3.k8s.com:2380 \
--initial-cluster-state new
```
etcd2
```
docker run -tid -p 2379-2380:2379-2380 \
--name etcd2 \
-h etcd2 \
-e TZ=UTC-8 \
-v /opt/etcd:/var/lib/etcd/ \
--restart=always \
hub.k8s.com/google-containers/etcd:3.1.11 \
etcd -name etcd2 \
--data-dir=/var/lib/etcd/etcd2.etcd \
--heartbeat-interval='100' \
--election-timeout='1000' \
--listen-client-urls=http://0.0.0.0:2379 \
--listen-peer-urls http://0.0.0.0:2380 \
--advertise-client-urls=http://etcd2.k8s.com:2379 \
--initial-advertise-peer-urls=http://etcd2.k8s.com:2380 \
--initial-cluster etcd1=http://etcd1.k8s.com:2380,etcd2=http://etcd2.k8s.com:2380,etcd3=http://etcd3.k8s.com:2380 \
--initial-cluster-state new
```
etcd3
```
docker run -tid -p 2379-2380:2379-2380 \
--name etcd3 \
-h etcd3 \
-e TZ=UTC-8 \
-v /opt/etcd:/var/lib/etcd/ \
--restart=always \
hub.k8s.com/google-containers/etcd:3.1.11 \
etcd -name etcd3 \
--data-dir=/var/lib/etcd/etcd3.etcd \
--heartbeat-interval='100' \
--election-timeout='1000' \
--listen-client-urls=http://0.0.0.0:2379 \
--listen-peer-urls http://0.0.0.0:2380 \
--advertise-client-urls=http://etcd3.k8s.com:2379 \
--initial-advertise-peer-urls=http://etcd3.k8s.com:2380 \
--initial-cluster etcd1=http://etcd1.k8s.com:2380,etcd2=http://etcd2.k8s.com:2380,etcd3=http://etcd3.k8s.com:2380 \
--initial-cluster-state new
```


验证
```
待续

```

## 单机部署 ##
如果你仅做测试，那可以使用单机部署
```
docker run -tid -p 2379-2380:2379-2380 \
--name etcd \
-h etcd1 \
-v /opt/etcd:/var/lib/etcd/ \
--restart=always \
hub.k8s.com/google-containers/etcd:3.1.11 \
etcd -name etcd \
--data-dir=/var/lib/etcd/etcd.etcd \
--election-timeout='1000' \
--listen-client-urls=http://0.0.0.0:2379 \
--listen-peer-urls http://0.0.0.0:2380 \
--advertise-client-urls=http://etcd1.k8s.com:2379 
```

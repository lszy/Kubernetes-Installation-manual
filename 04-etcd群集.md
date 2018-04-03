# 目的 #
使用k8s的staticPod部署etcd群集
# 需要主机 #
* etcd1.k8s.com
* etcd2.k8s.com
* etcd3.k8s.com
# 前提 #
已经安装flanneld、docker和kubelet
# 注意 #

本实验中，我们将在masters（etcds）.k8s.com上同时运行flannel和etcd，etcd要在flannel启动前进行相关部署。因为我们在容器中运行etcd。所以需要一下步骤进行初始化：
**步骤如下**
* 本实验中先启动位于master2和master3的etcd2和etcd3，这样etcd群集就可以正常工作。
* 启动etcd1(master1)的flanneld。工作正常后。"依次重启master2和master3的相关服务

# 群集部署 #
主机：etcd3.k8s.com

----------
**重要说明：**
在etcd3和etcd2 上安装flanneld、docker和kubelet后。不要使用systemctl自动启动任何一个程序。

----------

创建staticPod配置文件“etcd-cluster.yaml” 文件内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: etcd-cluster
  namespace: kube-system
  labels:
    name: etcd-cluster
spec:
  restartPolicy: Always
  hostNetwork: true
  containers:
  - name: etcd-cluster
    image: hub.k8s.com/google-containers/etcd:3.1.11
    command:
    - etcd
    - '--name=etcd3'
    - '--data-dir=/var/lib/etcd/etcd3.etcd'
    - '--heartbeat-interval=100'
    - '--election-timeout=500'
    - '--listen-client-urls=http://0.0.0.0:2379'
    - '--listen-peer-urls=http://0.0.0.0:2380'
    - '--advertise-client-urls=http://etcd3.k8s.com:2379'
    - '--initial-advertise-peer-urls=http://etcd3.k8s.com:2380'
    - '--initial-cluster="etcd1=http://etcd1.k8s.com:2380,etcd2=http://etcd2.k8s.com:2380,etcd3=http://etcd3.k8s.com:2380"'
    - '--initial-cluster-state=new'
    env:
    - name: TZ
      value: UTC-8
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 2379
        path: /health
      initialDelaySeconds: 15
      timeoutSeconds: 15
    securityContext:
      privileged: true
    volumeMounts:
    - name: data
      mountPath: /var/lib/etcd/
      readOnly: false
    ports:
    - name: etcd-service
      containerPort: 2379
      protocol: TCP
    - name: etcd-cluster
      containerPort: 2380
      protocol: TCP
  volumes:
  - name: data
    hostPath:
      path: /opt/etcd/
```
上面配置文件中，使用k8s启动了一个staticPod，pod中运行一个etcd服务，并且把数据文件挂载到了主机的/opt/etcd目录下。然后启动kubelet
````
shell(master3)># systemctl start kubelet
````
这时，etcd3的kubelet 会启动docker，并且加载etcd-cluster.yaml中的配置，启动staticPod etcd3，虽然没有flannel，但是我们看到容器正常启动。

如上所示,创建etcd2专属的配置文件后，启动etcd2.k8s.com的kubelet进程。这时，我们的etcd群集就可以正常工作了。

----------

* **如按照本示例说明，使用flanneld作为容器插件。为了我们之后可以正常启动flannel，我们需要向etcd注入数据，相关知识，请参照“04-A-容器网络-flannel”**
* 如不使用flannel，请继续，并自行忽略下例子中的flanneld部分。请替换成您使用的网络插件
* 示例中使用flannel作为网络组件。

----------
## 启动master1（etcd1）##
配置etcd1的配置文件，随后我们启动etcd1.k8s.com
````
shell(master1)># systemctl restart flanneld docker kubelet
````

**验证网络**
```
shell(master1)># ip addr | grep  -B 2 '10.254'

4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:45:d8:9f:5f brd ff:ff:ff:ff:ff:ff
    inet 10.254.22.129/25 brd 10.254.22.255 scope global docker0
--
11: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 9e:62:bd:d9:52:6f brd ff:ff:ff:ff:ff:ff
    inet 10.254.22.128/32 scope global flannel.1
```
**验证etcd群集**
````
shell(master1)># docker exec -ti `docker ps | grep -i 'name=etcd' | awk -F ' ' '{print $1}'`  etcdctl cluster-health
member 184beca37ca32d75 is healthy: got healthy result from http://etcd2.k8s.com:2379
member d79e9ae86b2a1de1 is healthy: got healthy result from http://etcd1.k8s.com:2379
member f7662e609b7e4013 is healthy: got healthy result from http://etcd3.k8s.com:2379
cluster is healthy
````
这样我们的etcd群集就启动成功了，现在我们需要重启master2(etcd3) 和 master3(etcd3) 来使其网络运行在flannel网络上。命令如下：
````
systemctl restart flanneld docker kubelet
````
验证如上述所述，不在说明。
# 单机部署 #
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

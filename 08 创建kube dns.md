# 目的 # 
创建k8s内部解析用dns
# 说明 #
这个文档较老，不过依旧可以使用。后期会更新
## 创建kube-dns 的rc和svc，文件为：kube-dns.yaml
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-dns-v8
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    version: v8
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 2
  selector:
    k8s-app: kube-dns
    version: v8
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        version: v8
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: kube2sky
        image: hub.k8s.com/google-containers/kube2sky:1.15
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
        args:
        # command = "/kube2sky"
        - --domain=cluster.local.
        - --etcd-server=http://etcd1.k8s.com:2379
        - --kube-master-url=http://master.k8s.com:8080
      - name: skydns
        image: hub.k8s.com/google-containers/skydns:2015-10-13-8c72f8c
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
        args:
        # command = "/skydns"
        - -machines=http://etcd1.k8s.com:2379
        - -addr=0.0.0.0:53
        - -domain=cluster.local.
        - -nameservers=10.10.1.252:53
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
      - name: healthz
        image: hub.k8s.com/google-containers/exechealthz:1.2
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
        args:
        - -cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1  >/dev/null
        - -port=8080
        ports:
        - containerPort: 8080
          protocol: TCP
      volumes:
      - name: etcd-storage
        emptyDir: {}
      dnsPolicy: Default  # Don't use cluster DNS.
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP:  10.254.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```
## 验证 ##
shell># kubectl  -n kube-system get pod -o wide | grep dns
```
kube-dns-v8-q9gwk             3/3       Running   0          41s       10.254.14.3     master1.k8s.com
kube-dns-v8-wl72s             3/3       Running   0          41s       10.254.20.131   master3.k8s.com
```
shell># kubectl  -n kube-system get svc -o wide | grep dns
```
kube-dns   ClusterIP   10.254.0.10   <none>        53/UDP,53/TCP   1m        k8s-app=kube-dns
```

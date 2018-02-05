因为apiserver 是无状态话的，所以可以使用简单的4层负载均衡实现高可用！下例子使用ha-proxy实现4层负载均衡和故障检查，使用keepalive实现虚拟地址

创建haproxy的配置文件
```
global
    log         127.0.0.1 local2
 
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    daemon
 
defaults
    mode                    tcp
    log                     global
    option                  abortonclose 
    option                  redispatch
    retries                 3
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    maxconn                 30000
 
listen stats
    mode http
    bind 0.0.0.0:1080
    stats enable
    stats hide-version
    stats uri     /haproxyadmin?stats
    stats realm   Haproxy\ Statistics
    stats auth    admin:123456
    stats admin if TRUE

listen check
    mode http
    bind 0.0:10200
    stats enable
    stats hide-version
    stats uri /healthz
    stats scope security_port 
 
frontend security_port
    bind *:8443
    mode tcp
    log global
    default_backend servers
 
 
backend servers
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server master1 10.10.1.21:6443 weight 1 check inter 1000 rise 2 fall 2
    server master2 10.10.1.22:6443 weight 1 check inter 1000 rise 2 fall 2
    server master3 10.10.1.23:6443 weight 1 check inter 1000 rise 2 fall 2

```
创建haproxy的staticPod文件
ha-proxy.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
  labels:
    name: haproxy
spec:
  restartPolicy: Always
  hostNetwork: true
  containers:
  - name: haproxy
    image: hub.k8s.com/apps/haproxy:1.8.3
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 10200
        path: /healthz
      initialDelaySeconds: 15
      timeoutSeconds: 15
    securityContext:
      privileged: true
    volumeMounts:
    - name: lib
      mountPath: /var/lib/haproxy
      readOnly: false
    - name: config
      mountPath: /usr/local/etc/haproxy/
      readOnly: false
    ports:
    - name: security-port
      containerPort: 8443
      protocol: TCP
    - name: check-port
      containerPort: 1080
      protocol: TCP
  volumes:
  - name: lib
    emptyDir: {}
  - name: config
    hostPath:
      path: /opt/haproxy/
```

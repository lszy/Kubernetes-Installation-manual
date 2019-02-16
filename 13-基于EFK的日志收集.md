# 目的 #
基于EFK实现日志收集功能,这里我们把elasticsearch和kibana做为Deployment部署。如果需要使用k8s群集外的elasticsearch和kibana。可跳过部署elasticsearch和kibana的阶段，然后使用deamonset（守护式进程）部署fluentd。
# 修改docker 日志模式 #
首先修改docker 的日志模式为json，对于CentOS使用的yum安装的的docker-1.12，我们需要使用如下进行配置。<br>
```
shell># Shell># vi /etc/sysconfig/docker
```
项|内容|备注
----|---|----
原始|OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
修改|OPTIONS='--selinux-enabled --signature-verification=false'

修改日志配置文件。如没有则创建
Shell>#vi /etc/docker/daemon.json
```
{
        "log-driver": "json-file",
        "log-opts": {
                "max-size": "10m",
                "max-file": "3"
        }
}
```
重启docker
shell># systemctl restar docker

# 部署 #
## 创建NameSpace ##
```
apiVersion: v1
kind: Namespace
metadata:
  name: kube-logging
```
## 部署elasticsearch ##
这里使用deployment部署elasticsearch单机应用，同时在k8s群集内发布服务。服务端口为9200和9300。
```
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-elasticsearch
  name: kubernetes-elasticsearch
  namespace: kube-logging
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-elasticsearch
  template:
    metadata:
      labels:
        k8s-app: kubernetes-elasticsearch
    spec:
      containers:
      - name: kubernetes-elasticsearch
        image: hub.k8s.com/apps/elasticsearch:6.2.3
        ports:
        - name: es-data
          containerPort: 9200
          protocol: TCP
        - name: es-cluster
          containerPort: 9300
          protocol: TCP
        volumeMounts:
        - mountPath: /usr/share/elasticsearch/data
          name: tmp-volume
      volumes:
      - name: tmp-volume
        emptyDir: {}

---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-elasticsearch
  name: elasticsearch
  namespace: kube-logging
spec:
  type: ClusterIP
  clusterIP: 10.254.0.202
  ports:
    - name: es-data
      port: 9200
      targetPort: 9200
    - name: es-cluster
      port: 9300
      targetPort: 9300
  selector:
    k8s-app: kubernetes-elasticsearch
```
## 部署kibana ##
### 创建kibana使用的configmap ##
这里创建kibana启动需要的配置的文件的configmap，在部署kibana时，我们映射到pod中。在配置文件中，定义了kibana的服务名称，服务主机地址，最重要的是，配置elasticsearch的服务路径和端口。注释部分为elasticsearch启用xpack模式后的安全特性，这里不适用。
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana
  namespace: kube-logging
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0"
    elasticsearch.url: http://elasticsearch.kube-logging.svc.cluster.local:9200
    #elasticsearch.username: elastic
    #elasticsearch.password: changeme
    #xpack.monitoring.ui.container.elasticsearch.enabled: true
```
### 创建kibana的deployment和service ###
创建名为kubernetes-kibana的deployment的部署，使用kibana 6.2.3部署一个pod，开放5601端口，同时挂载我们生成的名为“kibana”的configmap作为名为kibana.yml的配置文件。<br>
发布服务名为“kibana”的群集服务。
```
---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-kibana
  name: kubernetes-kibana
  namespace: kube-logging
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-kibana
  template:
    metadata:
      labels:
        k8s-app: kubernetes-kibana
    spec:
      containers:
      - name: kubernetes-elasticsearch
        image: hub.k8s.com/apps/kibana:6.2.3
        ports:
        - name: kibana-web
          containerPort: 5601
          protocol: TCP
        volumeMounts:
            - name: config
              mountPath: /usr/share/kibana/config/kibana.yml
              subPath: kibana.yml
      volumes:
      - name: config
        configMap:
          name: kibana
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-kibana
  name: kibana
  namespace: kube-logging
spec:
  type: ClusterIP
  clusterIP: 10.254.0.203
  ports:
    - name: kibana-web
      port: 5601
      targetPort: 5601
  selector:
    k8s-app: kubernetes-kibana

```
### 创建kibana的ingress配置 ###
说明：在执行下面命令前，需要先配置k8s的nginx ingress。可以参照《12-A-接入点-nginx ingress》。我们发布域名为kibana.k8s.com的虚拟主机为kibana的虚拟主机，内部服务为kibana，服务端口为5601.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: kube-logging
spec:
  rules:
  - host: kibana.k8s.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kibana
          servicePort: 5601
```
## 部署fluentd ##
### 创建RBAC ###
创建名为fluentd的服务账户，并赋予账户apiGroups的全部权限和获取pods资源，以及可以执行get、list和watch命令。
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-logging
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-logging
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-logging
```
### 创建ConfigMap ###
```
apiVersion: v1
kind: ConfigMap
metadata
  name: fluentd
  namespace: kube-logging
data:
  fluent.conf: |
    @include kubernetes.conf
    <match **>
       type elasticsearch
       log_level info
       include_tag_key true
       host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
       port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
       scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
       user "#{ENV['FLUENT_ELASTICSEARCH_USER']}"
       password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD']}"
       reload_connections "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'true'}"
       logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'logstash'}"
       logstash_format true
       buffer_chunk_limit 2M
       buffer_queue_limit 32
       flush_interval 5s
       max_retry_wait 30
       disable_retry_limit
       num_threads 8
    </match>
```
### 创建DaemonSet ###
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      containers:
        - name: fluentd
          image: hub.k8s.com/google-containers/fluentd-kubernetes-daemonset:elasticsearch
          env:
            - name:  FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.kube-logging.svc.cluster.local"
            - name:  FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            # X-Pack Authentication
            # =====================
            #- name: FLUENT_ELASTICSEARCH_USER
            #  value: "elastic"
            #- name: FLUENT_ELASTICSEARCH_PASSWORD
            #  value: "changeme"
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
            - name: config
              mountPath: /fluentd/etc/fluent.conf
              subPath: fluent.conf
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd
```

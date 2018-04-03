# 目的 #
基于EFK实现日志收集功能
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

# 部署elasticsearch 和kibana #
## 说明 ##
这部分仅作为示例。具体详见后续文档<br>部署elasticsearch
```
Shell># docker run -tid \
-h elasticsearch \
--name elasticsearch \
-p 9200:9200 \
-p 9300:9300 \
--restart=always  \
hub.k8s.com/apps/elasticsearch:5.5
```
部署kibana
```
Shell># docker run -tid \
-h kibana \
--name kibana \
--link elasticsearch:elasticsearch \
-p 5601:5601 \
--restart=always  \
hub.k8s.com/apps/kibana:5.5.2
```
# 部署 #
## 创建NameSpace ##
```
apiVersion: v1
kind: Namespace
metadata:
  name: kube-logging
```
## 创建RBAC ##
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
## 创建ConfigMap ##
```
apiVersion: v1
kind: ConfigMap
metadata:
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
## 创建DaemonSet ##
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
              value: "10.10.10.11"
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

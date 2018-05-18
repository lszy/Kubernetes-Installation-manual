# 目标 #
启动k8s master
# 说明 #
我们使用staticPod启动k8s master 的全部组件 前面我们已经使用staticPod启动了etcd群集，下面我们只需要将配置文件直接放到kubelet的staticPod目录中即可。

## 部署 ##
创建kube-master.yaml，并放到“/etc/kubernetes/kubelet.d/”
```
apiVersion: v1
kind: Pod
metadata:
  name: kube-master
  namespace: kube-system
  labels:
    name: kube-master
spec:
  restartPolicy: Always
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: hub.k8s.com/google-containers/kube-apiserver:v1.9.0
    command:
    - 'kube-apiserver'
    - '--bind-address=0.0.0.0'
    - '--insecure-bind-address=0.0.0.0'
    - '--secure-port=6443'
    - '--insecure-port=8080'
    - '--apiserver-count=3'
    - '--service-cluster-ip-range=10.254.0.0/16'
    - '--client-ca-file=/opt/kubernetes/pki/ca.pem'
    - '--service-account-key-file=/opt/kubernetes/pki/ca-key.pem'
    - '--tls-ca-file=/opt/kubernetes/pki/ca.pem'
    - '--tls-cert-file=/opt/kubernetes/pki/kubernetes.pem'
    - '--tls-private-key-file=/opt/kubernetes/pki/kubernetes-key.pem'
    - '--kubelet-certificate-authority=/opt/kubernetes/pki/ca.pem'
    - '--kubelet-client-certificate=/opt/kubernetes/pki/kubernetes.pem'
    - '--kubelet-client-key=/opt/kubernetes/pki/kubernetes-key.pem'
    - '--kubelet-https=true'
    - '--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds'
    - '--advertise-address=10.10.1.21'
    - '--authorization-mode=RBAC,Node'
    - '--allow-privileged=true'
    - '--etcd-servers=http://etcd1.k8s.com:2379,http://etcd2.k8s.com:2379,http://etcd3.k8s.com'
    securityContext:
      privileged: true
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 8080
        path: /healthz
      initialDelaySeconds: 15
      timeoutSeconds: 15
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: https
      containerPort: 6443
      protocol: TCP
    volumeMounts:
    - name: pki
      mountPath: /opt/kubernetes/pki/
      readOnly: false
  - name: kube-controller-manager
    image: hub.k8s.com/google-containers/kube-controller-manager:v1.9.0
    command:
    - 'kube-controller-manager'
    - '--address=0.0.0.0'
    - '--cluster-cidr=10.254.0.0/16 '
    - '--master=http://127.0.0.1:8080'
    - '--leader-elect=true'
    - '--root-ca-file=/opt/kubernetes/pki/ca.pem'
    - '--service-account-private-key-file=/opt/kubernetes/pki/ca-key.pem'
    - '--cluster-signing-cert-file=/opt/kubernetes/pki/ca.pem'
    - '--cluster-signing-key-file=/opt/kubernetes/pki/ca-key.pem'
    - '--logtostderr=true'
    - '--v=1'
    securityContext:
      privileged: false
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 10252
        path: /healthz
        initialDelaySeconds: 15
        timeoutSeconds: 15
    ports:
    - containerPort: 10252
      protocol: TCP
    volumeMounts:
    - name: pki
      mountPath: /opt/kubernetes/pki/
      readOnly: true
  - name: kube-scheduler
    image: hub.k8s.com/google-containers/kube-scheduler:v1.9.0
    command:
    - 'kube-scheduler'
    - '--address=0.0.0.0'
    - '--leader-elect=true'
    - '--master=http://127.0.0.1:8080'
    - '--logtostderr=true'
    - '--v=1'
    securityContext:
      privileged: false
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 10251
        path: /healthz
        initialDelaySeconds: 15
        timeoutSeconds: 15
    ports:
    - containerPort: 10251
      protocol: TCP
  volumes:
  - name: pki
    hostPath:
      path: /etc/kubernetes/pki/
```
上述文件中，需要将证书文件放到指定的主机目录中。
重启kubelet

## 验证 ##
shell># kubectl get cs
````
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
````

## 修改与kubelet的相关RBAC  ##
### 说明 ###
**下面的配置必须去做，否则我们的node因为权限不足无法创建pod.**
这里我们需要配置RBAC权限允许kubernetes API server 去接入每个节点的Kubelet API 去检索metrics, logs, 和Pod中运行的命令。<br>
我们将kubelet 的认证模式设置为webhook（--authorization-mode=webhook），Webhook模式使用SubjectAccessReview API来确定授权<br>
创建“system:kube-apiserver-to-kubelet” ClusterRole有权限访问Kubelet API，并执行与管理Pod相关最常见的任务：<br>
创建yaml文件
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
```
Kubernetes API 使用kubernetes用户去访问kubelet，使用由--kubelet-client-certificate标志定义的客户端证书进行身份验证。
将"system:kube-apiserver-to-kubelet" ClusterRole 绑定到“kubernetes:”用户。
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
```
赋予kubelet所在用户组访问API权限
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "watch", "list"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: "system:nodes"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```
创建CluseRoleBing “read-secrets-global” 并与“system:nodes”组绑定在一起，赋予system:nodes组内所有成员（所有node节点，单个用户名为：“system:node:Node1.k8s.com”）读取全部secrets的权限
##设置node节点role##
```
kubectl label node master1.k8s.com kubernetes.io/role=master
```
###验证###
```
kubectl get nodes -w
NAME              STATUS    ROLES     AGE       VERSION
master1.k8s.com   Ready     master    106d      v1.9.0
```
###特别说明###
特别感谢：http://www.recall704.com/cloud/k8s-node-roles/
通过代码，我们可以知道，k8s 把 label node-role.kubernetes.io/<role>="" 和 kubernetes.io/role="<role>" 来定义角色。
所以 master 可以这样
```
kubectl label node 192.168.88.201 node-role.kubernetes.io/master=true
```
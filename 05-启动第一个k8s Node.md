# 目的 #
 启动k8s node节点，并且使用staticPod运行kube-proxy
## 创建kube-proxy ##
### 生成kubeconfig ### 
```
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://master.k8s.com \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
*创建staticpod文件：kube-proxy.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
  labels:
    name: kube-proxy
spec:
  restartPolicy: Always
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: hub.k8s.com/google-containers/kube-proxy:v1.9.0
    command:
    - kube-proxy
    - --bind-address=0.0.0.0
    - --kubeconfig=/var/kube-proxy/kube-proxy.kubeconfig
    - --hostname-override=Master3.k8s.com
    - --cluster-cidr=10.254.0.0/16
    - --proxy-mode=iptables
    - --masquerade-all
    - --logtostderr=true
    - --v=2
    env:
    - name: TZ
      value: UTC-8
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 10256
        path: /healthz
      initialDelaySeconds: 15
      timeoutSeconds: 15
    securityContext:
      privileged: true
    volumeMounts:
    - name: dbus
      mountPath: /var/run/dbus
      readOnly: false
    - name: config
      mountPath: /var/kube-proxy
      readOnly: true
    ports:
    - containerPort: 10256
      protocol: TCP
  volumes:
  - name: dbus
    hostPath:
      path: /var/run/dbus
  - name: config
    hostPath:
      path: /etc/kubernetes/kube-proxy
```
我们将其保存并放入“/etc/kubernetes/kubelet.d/”中

## 启动第一个node节点 ##
因为我们的kube-apiserver，kube-controll-manager，kube-scheduler，kube-proxy都是用staticPod运行，所以我们需要启动第一个kubelet。<br>**本实例中使用master1.k8s.com**<br>我们的kubelet配置如下<br>
* 启动配置文件
shell># vi /etc/systemd/system/kubelet.service
```
Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
            $KUBELET_MAIN_ARGS \
            $KUBELET_NODE_NAME \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
* 创建kubeconfig文件
```
instance=`hostname`
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://master.k8s.com \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig

```

* 创建配置文件
shell># vi /etc/kubernetes/kubelet
```
KUBELET_MAIN_ARGS=" --address=0.0.0.0 --port=10250  --register-node --anonymous-auth=false  --authorization-mode=Webhook  --kubeconfig=/etc/kubernetes/Master1.k8s.com.kubeconfig --pod-infra-container-image=hub.k8s.com/google-containers/pause:3.0  --logtostderr=true --v=2 --allow-privileged=true"
KUBELET_NODE_NAME="--hostname-override=Master1.k8s.com"
KUBELET_ARGS="--cluster_dns=10.254.0.10 --cluster_domain=cluster.local --client-ca-file=/etc/kubernetes/pki/ca.pem --tls-private-key-file=/etc/kubernetes/pki/Master1.k8s.com-key.pem --tls-cert-file=/etc/kubernetes/pki/Master1.k8s.com.pem --pod-manifest-path=/etc/kubernetes/kubelet.d/"

```
* 重新载入<br>
Shell># systemctl daemon-reload
* 启动 <br>
Shell># systemctl start kubelet<br>
Shell># systemctl enable kubelet


 

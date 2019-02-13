# 目的 #

启动第一个kubelet节点

# 说明 #
因为我们用staticPod启动etcd、kube-apiserver、kube-controll-manager、kube-scheduler。所以我们需要优先配置kubelet。<br>
**本实例中使用master1.k8s.com**<br>
# 配置 kubelet #
## 启动配置文件 ##
shell># vi /etc/systemd/system/kubelet.service
```
【Unit]
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
## 创建kubeconfig文件 ##
这部分我们集成到“证书”章节中的kubelet中子章节中，在nodes.sh中自动生成。<br>
生成完成后，我们可以通过命令将生成的文件发送到节点服务器，以Node1.k8s.com为例。
```
shell># scp Node1.k8s.com* Node1.k8s.com:/etc/kubernetes/
```

## 创建配置文件 ##
shell># vi /etc/kubernetes/kubelet
```
KUBELET_MAIN_ARGS=" --address=0.0.0.0 --port=10250  --register-node --anonymous-auth=false  --authorization-mode=Webhook  --kubeconfig=/etc/kubernetes/Master1.k8s.com.kubeconfig --pod-infra-container-image=hub.k8s.com/google-containers/pause:3.0  --logtostderr=true --v=2 --allow-privileged=true"
KUBELET_NODE_NAME="--hostname-override=Master1.k8s.com"
KUBELET_ARGS="--cluster_dns=10.254.0.10 --cluster_domain=cluster.local --client-ca-file=/etc/kubernetes/pki/ca.pem --tls-private-key-file=/etc/kubernetes/pki/Master1.k8s.com-key.pem --tls-cert-file=/etc/kubernetes/pki/Master1.k8s.com.pem --pod-manifest-path=/etc/kubernetes/kubelet.d/"

```
## 重新载入 ##
```
Shell># systemctl daemon-reload
```
到此，我们结束kubelet的配置。 

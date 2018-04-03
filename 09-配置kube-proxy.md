# 目的 #
 使用DaemonSet部署运行kube-proxy
## 部署kube-proxy ##
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
# 创建 kube-proxy 使用的deamonset 文件
创建yaml文件：kube-proxy.yaml
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-proxy
  namespace: kube-system
  labels:
    k8s-app: kube-proxy
    version: 1.9.0
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: kube-proxy
        version: 1.9.0
        kubernetes.io/cluster-service: "true"
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      restartPolicy: Always
      hostNetwork: true
      containers:
      - name: kube-proxy
        image: hub.k8s.com/google-containers/kube-proxy:v1.9.0
        command:
        - kube-proxy
        - --bind-address=0.0.0.0
        - --kubeconfig=/var/kube-proxy/kube-proxy.kubeconfig
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
        - name: lib
          mountPath: /lib/modules
          readOnly: false
        ports:
        - containerPort: 10256
          protocol: TCP
      volumes:
      - name: dbus
        hostPath:
          path: /var/run/dbus
      - name: config
        secret:
          secretName: kube-proxy-kubeconfig
      - name: lib
        hostPath:
          path: /lib/modules
```
# 应用 #
我们使用kubectl应用配置
````
shell># kubectl create -f kube-proxy.yaml
````
# 验证 #
````
shell># kubectl describe  ds kube-proxy --namespace=kube-system
````
输出如下信息
````
Name:           kube-proxy
Selector:       k8s-app=kube-proxy,kubernetes.io/cluster-service=true,version=1.9.0
Node-Selector:  <none>
Labels:         k8s-app=kube-proxy
                kubernetes.io/cluster-service=true
                version=1.9.0
Annotations:    <none>
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  k8s-app=kube-proxy
           kubernetes.io/cluster-service=true
           version=1.9.0
  Containers:
   kube-proxy:
    Image:  hub.k8s.com/google-containers/kube-proxy:v1.9.0
    Port:   10256/TCP
    Command:
      kube-proxy
      --bind-address=0.0.0.0
      --kubeconfig=/var/kube-proxy/kube-proxy.kubeconfig
      --hostname-override=Master3.k8s.com
      --cluster-cidr=10.254.0.0/16
      --proxy-mode=iptables
      --masquerade-all
      --logtostderr=true
      --v=2
    Liveness:  http-get http://127.0.0.1:10256/healthz delay=15s timeout=15s period=10s #success=1 #failure=3
    Environment:
      TZ:  UTC-8
    Mounts:
      /lib/modules from lib (rw)
      /var/kube-proxy from config (ro)
      /var/run/dbus from dbus (rw)
  Volumes:
   dbus:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/dbus
    HostPathType:  
   config:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kube-proxy-kubeconfig
    Optional:    false
   lib:
    Type:          HostPath (bare host directory volume)
    Path:          /lib/modules
    HostPathType:  
Events:            <none>
````
结束
# 目的 #
访问k8s dashboard

**特别说明**
***
Kubernetes dashboard不应通过HTTP访问。对于通过HTTP访问的，将无法登录。单击登录页面上的登录按钮后将不会发生任何事情。
# 访问方式 # 
## kubectl proxy ##
## NodePort ##
**说明：**
如果是单机或者测试环境可以使用如下方案。
编辑 kubernetes-dashboard service。
```
shell># kubectl -n kube-system edit service kubernetes-dashboard
```
<table>
<thead>
	<td>标题</td>
	<td>内容</td>
</thead>
<tr>
	<td>修改前</td>
	<td><pre>
  apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
</pre></td>
</tr>
<tr>
	<td>修改后</td>
	<td><pre>
  apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30001
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
</pre></td>
</tr>
</table>

### 验证 ###
```
shell># kubectl get svc kubernetes-dashboard -n kube-system 
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.254.167.238   <none>        443:30001/TCP   104d
```
你可以访问kubernetes-dashboard所在节点的IP地址，如下
https://<node_ip>:30001/

## API Server ##
In case Kubernetes API server is exposed and accessible from outside you can directly access dashboard at: https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
Note: This way of accessing Dashboard is only possible if you choose to install your user certificates in the browser. In example certificates used by kubeconfig file to contact API Server can be used.
## Ingress ##
Dashboard可以使用ingress资源暴露. 请阅读<br>
https://kubernetes.io/docs/concepts/services-networking/ingress.
**说明**
我不会呢，望指教！


# 目的 # 
登录dashboard进行管理
# 简单用户 #
## 创建简单用户 ##
在本指南中，我们将了解如何创建一个使用Kubernetes服务帐户机制的新用户，授予该用户管理员权限登录使用bearer token（承载令牌）绑用户仪表板。
### 创建服务账户（SA） ###
我们在kube-system名称空间中创建一个“admin-user” 的SA
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
### 创建 ClusterRoleBinding ###
In most cases after provisioning our cluster using kops or kubeadm or any other popular tool admin Role already exists in the cluster. We can use it and create only RoleBinding for our ServiceAccount.

** 注意 ** ClusterRoleBinding 的 apiVersion 资源可能会因为k8s版本不同导致不一致。 
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
## 承载令牌(Bearer Token) ##
获取承载令牌
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
你会看到如下输出
```
Name:         admin-user-token-4w6sx
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=639c682b-372e-11e8-88b0-000c29f815af

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTR3NnN4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2MzljNjgyYi0zNzJlLTExZTgtODhiMC0wMDBjMjlmODE1YWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.AnIHuXbY0ktRzaZSw6ktjHuwIFj4NxQjSgRvbvbI6Gav1zO3F_E6sMWgp4AzpoC4BKbwCFNday146QLUqzFin9z5GoBKGnJ4AEvElYvfN5V1_zSbHpxf_1X1Hbbkl-O3G22GLO0g9c4-awULm5tr0NWXcU6pwVimDtjrWimO9KhTmWbKdt4ksz5LTebC1iznNTOt5F8y1LudsYeW7O12TUk1Kd7c31YiwIgzQDs9gEGIerry9QOOwNEe2aRelMSd3-CaTiqg2up0nF36AbdUpVQS-bSZQDIaDEqZuB8gKf8luk_-q9BZBPcByJidhy90vHHeGz5Iby21jPQJa54daQ
ca.crt:     1363 bytes

```
这里我们可以使用token中的值。来进行dashboard的登录操作。
## 验证 ##
<稍后补充>
# 使用kubeconfig 登录 #

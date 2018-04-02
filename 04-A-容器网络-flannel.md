# 目的 #
启动flannel网络

## 向etcd中注入数据 ##
在“etcd群集”章节中，我们已经说明了，etcd如何启动，现在我们说下注入数据
```
Shell>#curl -L http://etcd2.k8s.com:2379/v2/keys/flannel/network/config -XPUT -d value="{\"Network\":\"10.254.0.0/16\",\"SubnetLen\":25,\"Backend\":{\"Type\":\"vxlan\",\"VNI\":1}}"
```

## 配置flanneld ##
shell>#vi /etc/sysconfig/flanneld 
```
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://etcd1.k8s.com:2379,http://etcd2.k8s.com:2379,http://etcd3.k8s.com:2379;"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/flannel/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
```

# 目的 #
部署nginx-ingress
# 说明 #
文档存在不足，正在完善！ 
# default-http-backend #
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: hub.k8s.com/google-containers/defaultbackend:1.3
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels:
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
```
# 安装ningx-ingress-controller
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: nginx-ingress-lb
spec:
  replicas: 2
  selector:
    k8s-app: nginx-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-lb
        name: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - image: hub.k8s.com/google-containers/nginx-ingress-controller:0.9.0-beta.11
        name: nginx-ingress-lb
        imagePullPolicy: Always
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        # use downward API
        env:
          - name: KUBERNETES_MASTER
            value: http://master.k8s.com:8080
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        # we expose 18080 to access nginx stats in url /nginx-status
        # this is optional
        - containerPort: 18080
          hostPort: 18080
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
        - --apiserver-host=http://master.k8s.com:8080
```
# 验证 #
稍微完善

---
title: k8s集群部署prometheus服务
date: 2021-06-11 22:35:24
tags: k8s,prometheus
---

### prometheus介绍
> Prometheus 是一个开源系统监控和警报工具包，最初构建于 SoundCloud。Prometheus 是Kubernetes之后成为第二个正式加入CNCF基金会的项目。

### 部署
#### prometheus-ns.yaml
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: kube-mon
```

#### prometheus-rbac.yaml
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-mon
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
    - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-mon
```

#### prometheus-cm.yaml
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-mon
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
```

#### prometheus-deploy.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: kube-mon
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      securityContext:
        runAsUser: 0
      containers:
      - image: prom/prometheus:v2.14.0
        name: prometheus
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"  # 指定tsdb数据路径
        - "--storage.tsdb.retention.time=24h"
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        - "--web.console.libraries=/usr/share/prometheus/console_libraries"
        - "--web.console.templates=/usr/share/prometheus/consoles"
        ports:
        - containerPort: 9090
          name: http
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: config-volume
        - mountPath: "/prometheus"
          name: data
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      volumes:
      - name: data
        hostPath:
          path: /data/prometheus/
      - configMap:
          name: prometheus-config
        name: config-volume
```
> 部署之后发现报错
{% asset_img deploy-error.png 部署报错 %}

> 这样的错误信息，这是因为我们的 prometheus 的镜像中是使用的 nobody 这个用户，然后现在我们通过 hostPath 挂载到宿主机上面的目录的 ownership 却是 root

##### 解决办法（一）
> 这个时候我们就可以通过 securityContext 来为 Pod 设置下 volumes 的权限，通过设置 runAsUser=0 指定运行的用户为 root
```bash
...
securityContext:
  runAsUser: 0
```

#### prometheus-svc.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-mon
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: LoadBalancer #也可以使用NodePort通过节点IP直接访问
  ports:
    - name: web
      port: 9090
      targetPort: http
```

> 参考[Prometheus部署](https://www.qikqiak.com/k8strain/monitor/prometheus/)
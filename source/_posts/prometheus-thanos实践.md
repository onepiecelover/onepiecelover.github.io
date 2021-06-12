---
title: prometheus+thanos实践
date: 2021-06-11 18:07:54
tags: k8s,prometheus,thanos,S3
---

## 背景介绍
> 在以往的prometheus方案中，prometheus基本都是单例部署，如果以多副本的方式部署又会带来数据重复的问题，而且prometheus的数据都是本地磁盘存储，本地磁盘容量总归有上限，如果prometheus的数据量特别大，那么能查询到的监控数据的时间必定很短，在这样的背景下我们不得不寻找更优的解决方案。而thanos的出现解决了上述问题。

## thanos介绍
> 官方的介绍：Open source, highly available Prometheus setup with long term storage capabilities（是一款开源的，为Prometheus提供高可用和长期存储能力的解决方案）

## 部署实践
### 创建监控服务用的namespace
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: kube-mon
```

### prometheus部署
#### prometheus-rbac
```bash
# prometheus权限设置
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

#### prometheus-config
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-mon
data:
  prometheus.yaml.tmpl: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
      external_labels:
        cluster: prometheus-ha
        # Each Prometheus has to have unique labels.
        replica: $(POD_NAME)
    rule_files:
      - /etc/prometheus/rules/*rules.yaml
    alerting:
      # We want our alerts to be deduplicated
      # from different replicas.
      alert_relabel_configs:
      - regex: replica
        action: labeldrop
      alertmanagers:
        - scheme: http
          path_prefix: /
          static_configs:
            - targets: ['alertmanager:9093']
    scrape_configs:
    - job_name: kubernetes-nodes-cadvisor
      scrape_interval: 10s
      scrape_timeout: 10s
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2​
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
        - role: endpoints
      scheme: https 
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
        - role: endpoints
      relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: (.+)(?::\d+);(\d+)
          replacement: $1:$2
```

#### prometheus-rules.yaml
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: kube-mon
data:
  alert-rules.yaml: |-
    groups:
      - name: Deployment
        rules:
        - alert: Deployment at 0 Replicas
          annotations:
            summary: Deployment {{$labels.deployment}} in {{$labels.namespace}} is currently having no pods running
          expr: |
            sum(kube_deployment_status_replicas) by (deployment,namespace)  < 1
          for: 1m
          labels:
            team: node
      - name: Pods
        rules:
        - alert: Container restarted
          annotations:
            summary: Container named {{$labels.container}} in {{$labels.pod}} in {{$labels.namespace}} was restarted
          expr: |
            sum(increase(kube_pod_container_status_restarts_total[1m])) by (pod,namespace,container) > 0
          for: 0m
          labels:
            team: node
```

#### prometheus-sts
```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: kube-mon
  labels:
    app: prometheus
spec:
  serviceName: "prometheus"
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
      thanos-store-api: "true"
  template:
    metadata:
      labels: 
        app: prometheus
        thanos-store-api: "true"
    spec:
      securityContext:
        runAsUser: 0
      serviceAccountName: prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-rules
        configMap:
          name: prometheus-rules
      - name: prometheus-config-shared
        emptyDir: {}
      containers:
      - name: prometheus
        image: prom/prometheus:v2.14.0
        imagePullPolicy: IfNotPresent
        args:
        - "--config.file=/etc/prometheus-shared/prometheus.yaml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention.time=6h"
        - "--storage.tsdb.no-lockfile"
        - "--storage.tsdb.min-block-duration=2h"  # Thanos处理数据压缩
        - "--storage.tsdb.max-block-duration=2h"
        - "--web.enable-admin-api"  # 通过一些命令去管理数据
        - "--web.enable-lifecycle"  # 支持热更新  localhost:9090/-/reload 加载
        ports:
        - name: http
          containerPort: 9090
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "2Gi"
            cpu: "1"
        volumeMounts:
        - name: prometheus-config-shared
          mountPath: /etc/prometheus-shared/
        - name: prometheus-rules
          mountPath: /etc/prometheus/rules
        - name: data
          mountPath: "/prometheus"
      - name: thanos
        image: thanosio/thanos:v0.11.0
        imagePullPolicy: IfNotPresent
        args:
        - sidecar
        - --log.level=debug
        - --tsdb.path=/prometheus
        - --prometheus.url=http://localhost:9090
        - --reloader.config-file=/etc/prometheus/prometheus.yaml.tmpl
        - --reloader.config-envsubst-file=/etc/prometheus-shared/prometheus.yaml
        - --reloader.rule-dir=/etc/prometheus/rules/
        ports:
        - name: http-sidecar
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "2Gi"
            cpu: "2"
        volumeMounts:
        - name: prometheus-config-shared
          mountPath: /etc/prometheus-shared/
        - name: prometheus-config
          mountPath: /etc/prometheus
        - name: prometheus-rules
          mountPath: /etc/prometheus/rules
        - name: data
          mountPath: "/prometheus"
  volumeClaimTemplates:  # 由于prometheus每2h生成一个TSDB数据块，所以还是需要保存本地的数据
  - metadata:
      name: data
      labels:
        app: prometheus
    spec:
      storageClassName: ssd-csi-udisk #csi-ufile
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
```

#### prometheus-svc
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
  type: LoadBalancer
  ports:
    - name: web
      port: 9090
      targetPort: http
```
> 查询结果如下：
{% asset_img prometheus-sts.png 查询结果随机从一个实例中取 %}

#### headless
> 无头 Service 并不会分配 Cluster IP，kube-proxy 不会处理它们， 而且平台也不会为它们进行负载均衡和路由。 DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符。
```bash
# 该服务为查 querier 创建 srv 记录，以便查找 store-api 的信息
apiVersion: v1
kind: Service
metadata:
  name: thanos-store-gateway
  namespace: kube-mon
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: grpc
    port: 10901
    targetPort: grpc
  selector:
    thanos-store-api: "true" #匹配sts的pod，自动注册到dns中
```

### thanos部署
#### thanos-querier.yaml
```bash
# querier通过thanos-store-gateway去获取prometheus的数据
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-querier
  namespace: kube-mon
  labels:
    app: thanos-querier
spec:
  selector:
    matchLabels:
      app: thanos-querier
  template:
    metadata:
      labels:
        app: thanos-querier
    spec:
      containers:
      - name: thanos
        image: thanosio/thanos:v0.11.0
        args:
        - query
        - --log.level=debug
        - --query.replica-label=replica
        # Discover local store APIs using DNS SRV.
        - --store=dnssrv+thanos-store-gateway:10901
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "2Gi"
            cpu: "1"
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: http
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /-/healthy
            port: http
          initialDelaySeconds: 15
```

#### svc
```bash
apiVersion: v1
kind: Service
metadata:
  name: thanos-querier
  namespace: kube-mon
  labels:
    app: thanos-querier
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: http
    name: http
  selector:
    app: thanos-querier
  type: LoadBalancer
```

> 未去重结果
{% asset_img thanos-dup.png 未去重的结果 %}

> 去重结果
{% asset_img thanos-dedup.png 去重的结果 %}


### Store 组件
> Thanos now supported GCS, S3, Azure, Swift, Tencent COS and Aliyun OSS.所以对于一些暂时没有接入到官方库的对象存储产品，需要一个中转的服务：minio

#### minio
> MinIO’s high performance, Kubernetes-native object storage suite is
built for the demands of the hybrid cloud. Software-defined, it delivers
a consistent experience across every Kubernetes environment.(minio是一个云原生的对象存储服务)

### 部署minio
#### mini-namespace.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: minio
```

#### minio-deploy.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
  selector:
    matchLabels:
      app: minio 
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
      containers:
      - name: minio
        volumeMounts:
        - name: data 
          mountPath: /data
        image: minio/minio:RELEASE.2021-05-27T22-06-31Z
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
        readinessProbe:
          httpGet:
            path: /minio/health/ready
            port: 9000
          initialDelaySeconds: 90
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10
```

#### minio-pvc.yaml
> [在UK8S中使用US3](https://docs.ucloud.cn/uk8s/volume/ufile)
```bash
apiVersion: v1
kind: Secret
metadata:
  name: us3-secret
  namespace: kube-system
stringData:
  accessKeyID: TOKEN_9a6ec9fd-9cb7-4510-8ded-xxxxxxxx # 非账号公钥，为US3的令牌公钥。
  secretAccessKey: c429c8e5-e4e6-4366-bf93-xxxxxx # 非账号私钥，为US3的令牌私钥。
  endpoint: http://internal.s3-cn-bj.ufileos.com

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-ufile
provisioner: ufile.csi.ucloud.cn
parameters:
  bucket: s3-bucketname  # 事先申请好的US3 Bucket
  csi.storage.k8s.io/node-publish-secret-name: csi-s3-secret # 关联前一步创建的Secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio
spec:
  storageClassName: csi-ufile
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

#### minio-svc.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  type: LoadBalancer
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP
  selector:
    app: minio
```

### 关联thanos和minio
#### thanos-storage-minio.yaml
```bash
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objectstorage
  namespace: kube-mon
type: Opaque
stringData:
  thanos.yaml: |-
    type: s3
    config:
     bucket: prometheus
     endpoint: minio.minio.svc.cluster.local:9000
     access_key: minio
     insecure: true
     secret_key: minio123
```

#### store.yaml
```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-store-gateway
  namespace: kube-mon
  labels:
    app: thanos-store-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-store-gateway
  serviceName: thanos-store-gateway
  template:
    metadata:
      labels:
        app: thanos-store-gateway
        thanos-store-api: "true"
    spec:
      containers:
        - name: thanos
          image: thanosio/thanos:v0.21.0
          args:
          - "store"
          - "--log.level=debug"
          - "--data-dir=/data"
          - "--objstore.config-file=/etc/secret/thanos.yaml"
          - "--index-cache-size=500MB"
          - "--chunk-pool-size=500MB"
          ports:
          - name: http
            containerPort: 10902
          - name: grpc
            containerPort: 10901
          livenessProbe:
            httpGet:
              port: 10902
              path: /-/healthy
          readinessProbe:
            httpGet:
              port: 10902
              path: /-/ready
          volumeMounts:
            - name: object-storage-config
              mountPath: /etc/secret
              readOnly: false
      volumes:
        - name: object-storage-config
          secret:
            secretName: thanos-objectstorage
```

#### 修改sidecar的配置
> 我们需要把 objstore.config-file 参数和 Secret 对象也要配置到 Sidecar 组件中去。`volumes` `args` `volumeMounts`分别加上以下配置
```
......
volumes:
- name: object-storage-config
  secret:
    secretName: thanos-objectstorage
......
args:
...
- --reloader.rule-dir=/etc/prometheus/rules/
- --objstore.config-file=/etc/secret/thanos.yaml #添加该行
......
volumeMounts:
- name: object-storage-config
  mountPath: /etc/secret
  readOnly: false
......
```
> 配置生效过后正常的话就会有数据传入到 MinIO 里面去了，我们可以去 MinIO 的页面上查看验证

[参考thanos](https://www.qikqiak.com/k8strain/monitor/thanos/)

## 附录
#### node-export
```bash
piVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-mon
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: node-exporter
        image: prom/node-exporter:v0.18.1
        args:
        - --web.listen-address=$(HOSTIP):9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        ports:
        - containerPort: 9100
        env:
        - name: HOSTIP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            cpu: 150m
            memory: 180Mi
          limits:
            cpu: 150m
            memory: 180Mi
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
        volumeMounts:
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: root
          mountPath: /host/root
          mountPropagation: HostToContainer
          readOnly: true
      tolerations:
      - operator: "Exists"
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: dev
        hostPath:
          path: /dev
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

### 常用监控方案

#### Heapster

容器集群监控和性能分析工具，天然支持 kubernetes 和 CoreDns

#### Weave Scope

可以监控 kubernetes 集群中的一系列资源的状态、应用拓扑、scale，还可以通过浏览器进入容器内部调试，可实现基于 k8s 的 CI/CD 工作流

#### Prometheus

一套开源的时序数据库与监控告警系统，是目前主流的监控方案

### Prometheus

#### 0. 组件

**Prometheus server**，负责抓取和存储时间序列数据，通过服务发现或静态配置获取监控目标，是最重要的组件

**Pushgateway**，缓存短期任务指标(如 CronJob )，供 Promethus 拉取

**Exporters**，

**AlertManager**，



**Grafana**，提供数据可视化

#### 1. 自定义方式部署

##### 1.1 Prometheus ConfigMap 配置

```yaml
# prometheus-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitor

# k8s-prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']

    - job_name: 'kubernetes-nodes'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node

    - job_name: 'kubernetes-service'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: service

    - job_name: 'kubernetes-endpoints'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: endpoints

    - job_name: 'kubernetes-ingress'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: ingress

    - job_name: 'kubernetes-kubelet'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-cadvisor'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?:(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

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
      - target_label: __address__
        replacement: kubernetes.default.svc:443

    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.default.svc.cluster.local:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-ingresses'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: ingress
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_ingress_scheme, __address__, __meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.default.svc.cluster.local:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name
```

##### 1.2 Prometheus RBAC 配置

```yaml
# prometheus-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitor
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
  namespace: monitor
```

##### 1.3 Prometheus Server 配置

```yaml
# prometheus-server-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    name: prometheus
  namespace: monitor
spec:
  ports:
  - name: prometheus
    protocol: TCP
    port: 9090
    targetPort: 9090
  selector:
    app: prometheus
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: prometheus
  name: prometheus
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      serviceAccount: prometheus
      containers:
      - name: prometheus
        image: m.daocloud.io/docker.io/prom/prometheus:v2.53.4
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: prometheus-config
        - mountPath: "/etc/localtime"
          name: timezone
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: timezone
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
```

##### 1.4 Prometheus Exporter 配置

```yaml
# prometheus-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9100'
        prometheus.io/path: 'metrics'
      labels:
        app: node-exporter
      name: node-exporter
    spec:
      containers:
      - image: m.daocloud.io/docker.io/prom/node-exporter
        imagePullPolicy: IfNotPresent
        name: node-exporter
        ports:
        - containerPort: 9100
          name: scrape
      hostNetwork: true
      hostPID: true
```

##### 1.5 blackbox-exporter 配置

```yaml
# blackbox-exporter-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: monitor
spec:
  ports:
  - name: blackbox
    port: 9115
    protocol: TCP
  selector:
    app: blackbox-exporter
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - image: m.daocloud.io/docker.io/prom/blackbox-exporter
        imagePullPolicy: IfNotPresent
        name: blackbox-exporter
```

##### 1.6 Grafana 配置

```yaml
# grafana-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitor
  labels:
    app: grafana
    component: core
spec:
  type: NodePort
  ports:
  - port: 3000
    nodePort: 30011
  selector:
    app: grafana
    component: core
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: grafana-core
  namespace: monitor
  labels:
    app: grafana
    component: core
spec:
  serviceName: "grafana"
  selector:
    matchLabels:
      app: grafana
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
        component: core
    spec:
      containers:
      - image: m.daocloud.io/docker.io/grafana/grafana:6.5.3
        name: grafana-core
        imagePullPolicy: IfNotPresent
        env:
        - name: GF_AUTH_BASIC_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "false"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
        volumeMounts:
        - name: grafana-persistent-storage
          mountPath: /var/lib/grafana
          subPath: grafana
  volumeClaimTemplates:
  - metadata:
      name: grafana-persistent-storage
    spec:
      storageClassName: local-path
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

##### 1.7 访问 Prometheus 和 Grafana 服务

```shell
zhu@ubuntu-new:~/k8s/promethus$ kubectl get svc -n monitor
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
blackbox-exporter   ClusterIP   10.107.76.65   <none>        9115/TCP         124m
grafana             NodePort    10.98.61.60    <none>        3000:30011/TCP   124m
prometheus          NodePort    10.99.202.27   <none>        9090:32091/TCP   124m

# 集群内部访问
# Service 域名或 Service IP 均可
http://prometheus.monitor:9090

# 通过 NodePort 访问
http://<NodeIP>:30011
http://<NodeIP>:32091
```

#### 2. kube-promethus 部署

##### 2.1 部署

```shell
# 获取项目
git clone https://github.com/prometheus-operator/kube-prometheus

# 应用配置文件
kubectl apply --server-side -f manifests/setup
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f manifests/

# 移除部署
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```

##### 2.2 配置 Ingress

```shell
# 获取部署的 Service
kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.111.8.136     <none>        9093/TCP,8080/TCP            3h34m
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   3h32m
blackbox-exporter       ClusterIP   10.105.237.221   <none>        9115/TCP,19115/TCP           3h34m
grafana                 ClusterIP   10.104.207.159   <none>        3000/TCP                     3h34m
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            3h34m
node-exporter           ClusterIP   None             <none>        9100/TCP                     3h34m
prometheus-adapter      ClusterIP   10.109.149.150   <none>        443/TCP                      3h34m
prometheus-k8s          ClusterIP   10.109.155.44    <none>        9090/TCP,8080/TCP            3h34m
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     3h32m
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     3h34m
```

```yaml
# prometheus-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: monitoring
  name: prometheus-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.zhu.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
  - host: prometheus.zhu.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090
  - host: alertmanager.zhu.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: alertmanager-main
            port:
              number: 9093
```

```shell
# 设置 /etc/hosts

# 验证可访问性
curl grafana.zhu.cn
curl prometheus.zhu.cn
curl alertmanager.zhu.cn

# 通过浏览器访问对应地址, 若无法访问, 检查防火墙或对应 VPN 设置
```

### ELK

#### 0. 简介

日志收集和分析领域广泛使用的技术栈，由三个核心开源组件组成：Elasticsearch，Logstash，Kibana。随着技术演进，部分场景会引入 Filebeat 等轻量级工具代替 Logstash。

**Elasticsearch**	日志搜索

**LogStash**		数据清洗

**Filebeat**  		数据采集

**Kibana**    		可视化界面

#### 1. Elasticsearch

**存储与搜索**，作为分布式搜索引擎，负责日志的存储、索引和实时检索。

- 分布式架构：自动分片和副本机制，支持高可用与水平扩展
- 全文搜索：基于 Apache Lucene 实现倒排索引，支持海量数据的快速查询与分析
- 多数据类型支持：兼容结构化、非结构化数据及地理空间数据

##### 1.1 部署

```yaml
# es-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elasticsearch-logging
  namespace: logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: elasticsearch-logging
  namespace: logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "services"
  - "namespaces"
  - "endpoints"
  verbs:
  - "get"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: elasticsearch-logging
  namespace: logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: elasticsearch-logging
  namespace: logging
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: elasticsearch-logging
  apiGroup: ""
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-logging
  namespace: logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Elasticsearch"
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    k8s-app: elasticsearch-logging
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-logging
  namespace: logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    srv: srv-elasticsearch
spec:
  serviceName: elasticsearch-logging
  replicas: 1
  selector:
    matchLabels:
      k8s-app: elasticsearch-logging
  template:
    metadata:
      labels:
        k8s-app: elasticsearch-logging
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccountName: elasticsearch-logging
      nodeSelector:
        es: data
      tolerations:
      - key: ""
        operator: Exists
        effect: NoSchedule
      initContainers:
      - name: elasticsearch-logging-init
        image: m.daocloud.io/docker.io/alpine:3.6
        command: ["/sbin/sysctl", "-w", "vm.max_map_count=1048576"]
        securityContext:
          privileged: true
      - name: incress-fd-ulimit
        image: m.daocloud.io/docker.io/busybox
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      - name: elasticsearch-volume-init
        image: m.daocloud.io/docker.io/alpine:3.6
        command: ["chmod", "-R", "777", "/usr/share/elasticsearch/data"]
        volumeMounts:
        - name: elasticsearch-logging
          mountPath: /usr/share/elasticsearch/data
      containers:
      - image: m.daocloud.io/docker.io/elasticsearch:7.9.3
        name: elasticsearch-logging
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 500Mi
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: elasticsearch-logging
          mountPath: /usr/share/elasticsearch/data/
        env:
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: "discovery.type"
          value: "single-node"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx2g"
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-logging
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-path
```

#### 2. Logstash

**采集与处理**，日志处理管道，负责从多源(如文件、数据库、消息队列)采集数据，并完成过滤、转换后输出至 Elasticsearch。

- 输入插件：支持文件(File)、系统日志(Syslog)、Beats 等多种数据源
- 过滤插件：如 Grok 解析非结构化日志、GeoIP 解析地理位置等
- 输出插件：将处理后的日志发送至 Elasticsearch、kafka 等目标

##### 2.1 部署

```yaml
# logstash-deployment.yaml
apiVersion: v1 
kind: Service 
metadata: 
  name: logstash 
  namespace: logging 
spec: 
  ports: 
  - port: 5044 
    targetPort: beats 
  selector: 
    type: logstash 
  clusterIP: None 
--- 
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: logstash 
  namespace: logging 
spec: 
  selector: 
    matchLabels: 
      type: logstash 
  template: 
    metadata: 
      labels: 
        type: logstash 
        srv: srv-logstash 
    spec: 
      containers: 
      - image: m.daocloud.io/docker.io/logstash:7.9.3 #该镜像支持arm64和amd64两种架构 
        name: logstash 
        ports: 
        - containerPort: 5044 
          name: beats 
        command: 
        - logstash 
        - '-f' 
        - '/etc/logstash_c/logstash.conf' 
        env: 
        - name: "XPACK_MONITORING_ELASTICSEARCH_HOSTS" 
          value: "http://elasticsearch-logging:9200" 
        volumeMounts: 
        - name: config-volume 
          mountPath: /etc/logstash_c/ 
        - name: config-yml-volume 
          mountPath: /usr/share/logstash/config/ 
        - name: timezone 
          mountPath: /etc/localtime 
        resources: #logstash一定要加上资源限制，避免对其他业务造成资源抢占影响 
          limits: 
            cpu: 1000m 
            memory: 2048Mi 
          requests: 
            cpu: 512m 
            memory: 512Mi 
      volumes: 
      - name: config-volume 
        configMap: 
          name: logstash-conf 
          items: 
          - key: logstash.conf 
            path: logstash.conf 
      - name: timezone 
        hostPath: 
          path: /etc/localtime 
      - name: config-yml-volume 
        configMap: 
          name: logstash-yml 
          items: 
          - key: logstash.yml 
            path: logstash.yml 
 
--- 
apiVersion: v1 
kind: ConfigMap 
metadata: 
  name: logstash-conf 
  namespace: logging 
  labels: 
    type: logstash 
data: 
  logstash.conf: |- 
    input {
      beats { 
        port => 5044 
      } 
    } 
    filter {
      # 处理 ingress 日志 
      if [kubernetes][container][name] == "nginx-ingress-controller" {
        json {
          source => "message" 
          target => "ingress_log" 
        }
        if [ingress_log][requesttime] { 
          mutate { 
            convert => ["[ingress_log][requesttime]", "float"] 
          }
        }
        if [ingress_log][upstremtime] { 
          mutate { 
            convert => ["[ingress_log][upstremtime]", "float"] 
          }
        } 
        if [ingress_log][status] { 
          mutate { 
            convert => ["[ingress_log][status]", "float"] 
          }
        }
        if  [ingress_log][httphost] and [ingress_log][uri] {
          mutate { 
            add_field => {"[ingress_log][entry]" => "%{[ingress_log][httphost]}%{[ingress_log][uri]}"} 
          } 
          mutate { 
            split => ["[ingress_log][entry]","/"] 
          } 
          if [ingress_log][entry][1] { 
            mutate { 
              add_field => {"[ingress_log][entrypoint]" => "%{[ingress_log][entry][0]}/%{[ingress_log][entry][1]}"} 
              remove_field => "[ingress_log][entry]" 
            }
          } else { 
            mutate { 
              add_field => {"[ingress_log][entrypoint]" => "%{[ingress_log][entry][0]}/"} 
              remove_field => "[ingress_log][entry]" 
            }
          }
        }
      }
      # 处理以srv进行开头的业务服务日志 
      if [kubernetes][container][name] =~ /^srv*/ { 
        json { 
          source => "message" 
          target => "tmp" 
        } 
        if [kubernetes][namespace] == "kube-logging" { 
          drop{} 
        } 
        if [tmp][level] { 
          mutate{ 
            add_field => {"[applog][level]" => "%{[tmp][level]}"} 
          } 
          if [applog][level] == "debug"{ 
            drop{} 
          } 
        } 
        if [tmp][msg] { 
          mutate { 
            add_field => {"[applog][msg]" => "%{[tmp][msg]}"} 
          } 
        } 
        if [tmp][func] { 
          mutate { 
            add_field => {"[applog][func]" => "%{[tmp][func]}"} 
          } 
        } 
        if [tmp][cost]{ 
          if "ms" in [tmp][cost] { 
            mutate { 
              split => ["[tmp][cost]","m"] 
              add_field => {"[applog][cost]" => "%{[tmp][cost][0]}"} 
              convert => ["[applog][cost]", "float"] 
            } 
          } else { 
            mutate { 
              add_field => {"[applog][cost]" => "%{[tmp][cost]}"} 
            }
          }
        }
        if [tmp][method] { 
          mutate { 
            add_field => {"[applog][method]" => "%{[tmp][method]}"} 
          }
        }
        if [tmp][request_url] { 
          mutate { 
            add_field => {"[applog][request_url]" => "%{[tmp][request_url]}"} 
          } 
        }
        if [tmp][meta._id] { 
          mutate { 
            add_field => {"[applog][traceId]" => "%{[tmp][meta._id]}"} 
          } 
        } 
        if [tmp][project] { 
          mutate { 
            add_field => {"[applog][project]" => "%{[tmp][project]}"} 
          }
        }
        if [tmp][time] { 
          mutate { 
            add_field => {"[applog][time]" => "%{[tmp][time]}"} 
          }
        }
        if [tmp][status] { 
          mutate { 
            add_field => {"[applog][status]" => "%{[tmp][status]}"} 
            convert => ["[applog][status]", "float"] 
          }
        }
      }
      mutate { 
        rename => ["kubernetes", "k8s"] 
        remove_field => "beat" 
        remove_field => "tmp" 
        remove_field => "[k8s][labels][app]" 
      }
    }
    output { 
      elasticsearch { 
        hosts => ["http://elasticsearch-logging:9200"] 
        codec => json 
        index => "logstash-%{+YYYY.MM.dd}" #索引名称以logstash+日志进行每日新建 
      }
    } 
---
apiVersion: v1 
kind: ConfigMap 
metadata: 
  name: logstash-yml 
  namespace: logging 
  labels: 
    type: logstash 
data: 
  logstash.yml: |- 
    http.host: "0.0.0.0" 
    xpack.monitoring.elasticsearch.hosts: http://elasticsearch-logging:9200
```

#### 3. Kibana

**可视化监控**，提供 Web 界面，用于日志的可视化展示与交互式分析。

- 仪表盘定制：通过图表、热力图等形式展示日志趋势
- 查询分析：使用 KQL(Kibana Query Language) 进行复杂数据筛选
- 告警功能：基于日志阈值设置实时告警

##### 3.1 部署

```yaml
# kibana-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: logging
  name: kibana-config
  labels:
    k8s-app: kibana
data:
  kibana.yaml: |-
    server.name: kibana
    server.host: "0"
    i18n.locale: zh-CN
    elasticsearch:
      hosts: ${ELASTICSEARCH_HOSTS}
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    k8s-app: kibana
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Kibana"
    srv: srv-kibana
spec:
  type: NodePort
  ports:
  - port: 5601
    protocol: TCP
    targetPort: ui
  selector:
    k8s-app: kibana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    k8s-app: kibana
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Kibana"
    srv: srv-kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: kibana
  template:
    metadata:
      labels:
        k8s-app: kibana
    spec:
      containers:
      - name: kibana
        image: m.daocloud.io/docker.io/kibana:7.9.3
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch-logging:9200
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yaml
          readOnly: true
          subPath: kibana.yaml
      volumes:
      - name: config
        configMap:
          name: kibana-config
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
spec:
  ingressClassName: nginx
  rules:
  - host: kibana.zhu.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
```

#### 4. Filebeat 或 Fluentd

**EFK架构**，在资源受限或云原生环境中，常用 Filebeat 或 Fluentd 替代 Logstash

##### 4.1 Filebeat 部署

```yaml
# filebeat-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
  labels:
    k8s-app: filebeat
data:
  filebeat.yaml: |-
    filebeat.inputs:
    - type: container
      enable: true
      paths:
      - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchars:
          - logs_path:
            logs_path: "/var/log/containers/"
    output.logstash:
      hosts: ["logstash:5044"]
      enable: true
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: logging
  labels:
    k8s-app: filebeat
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: m.daocloud.io/docker.io/elastic/filebeat:7.9.3
        args:
        - -c
        - /etc/filebeat.yaml
        - -e
        - -httpprof
        - 0.0.0.0:6060
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-logging
        - name: ELASTICSEARCH_PORT
          value: "9200"
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 1000Mi
            cpu: 1000m
          requests:
            memory: 100Mi
            cpu: 100m
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yaml
          readOnly: true
          subPath: filebeat.yaml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib
          readOnly: true
        - name: timezone
          mountPath: /etc/localtime
        - name: varlog
          mountPath: /var/log/
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib
      - name: varlog
        hostPath:
          path: /var/log/
      - name: inputs
        configMap:
          defaultMode: 0600
          name: filebeat-inputs
      - name: data
        hostPath:
          path: /data/filebeat-data
          type: DirectoryOrCreate
      - name: timezone
        hostPath:
          path: /etc/localtime
      tolerations:
      - key: dedicated
        value: gpu
        operator: Equal
        effect: NoExecute
      - effect: NoSchedule
        operator: Exists
```


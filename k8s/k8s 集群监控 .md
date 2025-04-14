### 常用监控方案

#### Heapster

容器集群监控和性能分析工具，天然支持 kubernetes 和 CoreDns. 

#### Weave Scope

可以监控 kubernetes 集群中的一系列资源的状态、应用拓扑、scale，还可以通过浏览器进入容器内部调试，可实现基于 k8s 的 CI/CD 工作流

#### Promethus

一套开源的时序数据库与监控告警系统，是目前主流的监控方案. 

### Promethus

#### 0. 组件

**Promethus server**，负责抓取和存储时间序列数据，通过服务发现或静态配置获取监控目标，是最重要的组件

**Pushgateway**，缓存短期任务指标(如 CronJob )，供 Promethus 拉取

**Exporters**，

**AlertManager**，



**Grafana**，提供数据可视化

#### 1. 自定义部署

##### 1.1 ConfigMap 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promethus-config
  namespace: monitor
data:
  promethus.yml: |
    global:
      scrape_intervals: 15s
      evaluation_intervals: 15s
    scape_configs:
    - job_name: 'promethus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'kubernetes-nodes'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_config:
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
         replacement: /api/vi/nodes/${1}/proxy/metrics/cadvisor
       - action: label_map
         regex: __meta_kubernetes_node_label_(.+)
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_promethus_to_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_promethus_io_path]
          action: replace
          target_label: __metrics_path__
          regix: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_promethus_io_port]
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
        - souece_labels: [__meta_kubernetes_service_annotation_promethus_io_probe]
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
      - job_name: 'kubernetes-ingress'
        metrics_path: /probe
        params:
          module: [http_2xx]
        kubernetes_sd_configs:
        - role: ingress
        relabel_configs:
        - source_labels: [__meta_kubernetes_ingress_annotation_promethus_io_probe]
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



#### 2. kube-promethus 部署
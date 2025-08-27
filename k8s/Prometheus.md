## Guides

### 快速上手

参考[链接](https://prometheus.io/docs/introduction/first_steps/).

#### 1. 准备 Prometheus

```shell
# 下载并解压对应软件包
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

#### 2. 配置 Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"
```

#### 3. 启动 Prometheus

```shell
./prometheus --config.file=prometheus.yml

# 启动后访问 http://localhost:9090

# 核实 Prometheus 正在向外提供自身指标，访问 http://localhost:9090/metrics

# 重新加载配置文件
kill -s SIGHUP <PID>
curl -X POST http://localhost:9090/-/reload
```

#### 4. 使用表达式浏览器和图形接口

参考[链接](https://prometheus.io/docs/introduction/first_steps/#using-the-expression-browser).

### 基于 Basic Auth 加密 Prometheus 

#### 1.

### Node Exporter 监视 Linux 指标

Prometheus Node Exporter 可以暴露关于硬件和内核相关的大量指标. 参考[链接](https://prometheus.io/docs/guides/node-exporter/).

#### 1. 运行 Node Exporter

```shell
# NOTE: Replace the URL with one from the above mentioned "downloads" page.
# <VERSION>, <OS>, and <ARCH> are placeholders.
# "downloads" page: https://prometheus.io/download/#node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v<VERSION>/node_exporter-<VERSION>.<OS>-<ARCH>.tar.gz
tar xvfz node_exporter-*.*-amd64.tar.gz
cd node_exporter-*.*-amd64
./node_exporter

# 核实指标已暴露
curl http://localhost:9100/metrics
```

#### 2. 配置 Prometheus 实例

```yaml
global:
  scrape_interval: 15s

scrape_configs:
- job_name: node-k8s-master0
  static_configs:
  - targets: ['192.168.0.110:9100']
  
# 配置后重新启动 prometheus
./prometheus --config.file=./prometheus.yml

# 通过 Prometheus 表达式浏览器浏览 Node Exporter 监测到的指标

# 添加多个 Node Exporter
scrape_configs:
  - job_name: 'node'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

### 抓取 docker 相关指标

#### 1. cAdvisor compose 文件

```yaml
services:
  srv-cadvisor:
    image: m.daocloud.io/gcr.io/cadvisor/cadvisor
    ports:
    - target: 8080
      published: 8080
      mode: host # bypass the routing mesh
    labels:
      prometheus-job: cadvisor
    deploy:
      mode: global
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker:/var/lib/docker:ro
    command: --docker_only
```

#### 2. 配置抓取项

```yaml
# scrape-config.yaml  
  
  # Create a job for Docker Swarm containers.
#  - job_name: "docker-containers"
#    docker_sd_configs:
#      - host: unix:///var/run/docker.sock # You can also use http/https to connect to the Docker daemon.

  - job_name: 'node-swarm'
    dockerswarm_sd_configs:
      - host: unix:///var/run/docker.sock
        role: nodes
   #     port: 9323
    relabel_configs:
      # Fetch metrics on port 9323.
      - source_labels: [__meta_dockerswarm_node_address]
        target_label: __address__
        replacement: $1:9323
      # Set hostname as instance label
      - source_labels: [__meta_dockerswarm_node_hostname]
        target_label: instance
        replacement: $1-swarm

  # Create a job for Swarm tasks.
  - job_name: 'task-swarm'
    dockerswarm_sd_configs:
      - host: unix:///var/run/docker.sock
        role: tasks
    relabel_configs:
      # Set hostname as instance label
      - source_labels: [__meta_dockerswarm_node_hostname]
        target_label: instance
        replacement: $1-cadvisor
      - source_labels: [__meta_dockerswarm_node_address]
        target_label: __address__
        replacement: $1:8080
      # Only keep containers that have a `prometheus-job` label.
      - source_labels: [__meta_dockerswarm_container_label_prometheus_job]
        regex: .+
        action: keep
#      - source_labels: [__meta_dockerswarm_node_role]
#        regex: manager
#        action: keep
      # Use the prometheus-job Swarm label as Prometheus job label.
      - source_labels: [__meta_dockerswarm_container_label_prometheus_job]
        target_label: job
        replacement: $1

  # Create a job for Swarm services.
#  - job_name: 'service-swarm'
#    dockerswarm_sd_configs:
#      - host: unix:///var/run/docker.sock
#        role: services
```

## 概念

### 数据模型

Prometheus 从根本上将所有数据存储为时间序列：同一个指标下的相同标签维度的带有时间戳的数值流. 除此之外，Prometheus 也可生成基于临时查找结果的经过计算推导的时间序列. 

#### 指标名称和标签

时间序列由其指标名称和可选的标签键值对进行区分. 

**指标名称**

- 指标名称应该指出测量系统的通用特征(例如 `http_requests_total` 指接收到的 HTTP 请求总数)
- 指标名称应使用 UTF-8 字符集
- 指标名称应满足特定的正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*`，不满足该表达式的名称在使用 PromQL 时需要引号扩起. (冒号被预留用作用户自定义的记录规则使用，不应该被 exporters 或 direct instrumentation 使用. )

**指标标签**

标签可以表征相同指标名称下的不同实例. 在 Prometheus 中这被称为维度数据模型. 查询语言基于这些维度进行数据的筛选和聚合. 对于标签的改动，无论是新增标签还是移除标签，都会产生新的时间序列. 

- 标签的键和值应该使用 UTF-8 字符集
- 标签名称由 `__` 开头的是作为 Prometheus 内部预留使用
- 指标名称应满足特定的正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*`，不满足该表达式的名称在使用 PromQL 时需要引号扩起.  
- 具有空值的标签被视作不存在的标签

#### 数据样式

时序数据的样本由以下两部分组成

- 一个 float64 或 native histogram 值
- 一个毫秒精度的时间戳

#### 表示形式

对给定的指标名称和标签集合，时间序列通常由以下表示形式进行区分：

`<metric_name>{<label_name>="<label value>", ...}`

对于为按规则定义的名称需要使用引号扩起，并使用以下表示形式：

`{"<metric_name>", <label_name>="<label value>", ...}`  或

`{__name__="<metric_name>", <label_name>="<label value>", ...}`

### 指标类型

参考[链接](https://prometheus.io/docs/concepts/metric_types/).

Prometheus 客户端库提供四种核心的指标类型. 这些类型目前仅在不同的客户端库中存在差异. Prometheus 服务器目前还没有利用这些类型信息并将所有数据都看作为无类型的时间序列，这一点后续可能会有所改变.

#### Counter

表征单调递增计数器的累加指标，其值只能递增或在重置时被设为0. 

#### Gauge

表征可以任意增减的单个数值. 

#### Histogram

直方图用于统计样本分布的高级数据类型，特别适用于分析请求延迟、响应大小等需要观察数据分布的场景. 

#### Summary

与 histogram 类似，...

### 工作和实例

在 Prometheus 中，可以抓取的一个端点被称为实例，通常与一个单一进程对应. 相同目的的实例集合，或是为了伸缩和可靠性而重复的相同进程，被称为工作

#### 标签和时间序列的自动生成

当 Prometheus 抓取目标时，它会在抓取时自动为其打上一些标签以区分这些目标. 

- job：这个目标属于的 job 名称
- instance：抓取目标的 URL 

若以上标签已经存在于被抓取的数据当中时，Prometheus 的行为将取决与 `honor_labels` 选项的配置情况. 

**对每一个抓取的实例，Prometheus 会自动存储以下时间序列：**

- `up{job="<job-name>", instance="<instance-id>"}`，值为 1 表示实例是健康的或是可获取的，为 0 表示抓取失败
- `scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}`，值为设置的抓取持续时间
- `scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}`，经过重标记处理后保留下来的有效样本数
- `scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}`，目标暴露的样本数
- `scrape_series_added{job="<job-name>", instance="<instance-id>"}`，新抓取到时许数列的新增样本数，若该值突增说明应用新增了动态标签或配置错误，若该值稳定为 0，说明时序集合稳定. 

**若启用了 `extra-scrape-metrics` 特征标志，则将额外生成以下的指标：**

- `scrape_timeout_seconds{job="<job-name>", instance="<instance-id>"`，配置的 `scrape_timeout`
- `scrape_sample_limit{job="<job-name>", instance="<instance-id>"`，为目标配置的 `sample_limit` 值，为 0 表示为配置限值
- `scrape_body_size_bytes{job="<job-name>", instance="<instance-id>"`，最近一次抓取结果的未压缩状态下的大小，若因 `body_size_limit` 抓取失败为 -1，其他情况的抓取失败为 0.

## 配置

Prometheus 可以在运行时重新加载配置，若新的配置文件无效，则不会应用改动. 可以通过向 Prometheus 进程发送 SIGHUP 信号或向 ~/reload URL 端口发送 HTTP POST 请求(当指定 `--web.enable-lifecycle` 选项时)来重新加载配置以及规则文件. 

### Configuration File

配置文件定义抓取参数，参考[链接](https://prometheus.io/docs/prometheus/latest/configuration/configuration/). 

```yaml
global:
  # How frequently to scrape targets by default.
  [ scrape_interval: <duration> | default = 1m ]

  # How long until a scrape request times out.
  # It cannot be greater than the scrape interval.
  [ scrape_timeout: <duration> | default = 10s ]

  # The protocols to negotiate during a scrape with the client.
  # Supported values (case sensitive): PrometheusProto, OpenMetricsText0.0.1,
  # OpenMetricsText1.0.0, PrometheusText0.0.4.
  # The default value changes to [ PrometheusProto, OpenMetricsText1.0.0, OpenMetricsText0.0.1, PrometheusText0.0.4 ]
  # when native_histogram feature flag is set.
  [ scrape_protocols: [<string>, ...] | default = [ OpenMetricsText1.0.0, OpenMetricsText0.0.1, PrometheusText0.0.4 ] ]

  # How frequently to evaluate rules.
  [ evaluation_interval: <duration> | default = 1m ]

  # Offset the rule evaluation timestamp of this particular group by the
  # specified duration into the past to ensure the underlying metrics have
  # been received. Metric availability delays are more likely to occur when
  # Prometheus is running as a remote write target, but can also occur when
  # there's anomalies with scraping.
  [ rule_query_offset: <duration> | default = 0s ]

  # The labels to add to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager). 
  # Environment variable references `${var}` or `$var` are replaced according 
  # to the values of the current environment variables. 
  # References to undefined variables are replaced by the empty string.
  # The `$` character can be escaped by using `$$`.
  external_labels:
    [ <labelname>: <labelvalue> ... ]

  # File to which PromQL queries are logged.
  # Reloading the configuration will reopen the file.
  [ query_log_file: <string> ]

  # File to which scrape failures are logged.
  # Reloading the configuration will reopen the file.
  [ scrape_failure_log_file: <string> ]

  # An uncompressed response body larger than this many bytes will cause the
  # scrape to fail. 0 means no limit. Example: 100MB.
  # This is an experimental feature, this behaviour could
  # change or be removed in the future.
  [ body_size_limit: <size> | default = 0 ]

  # Per-scrape limit on the number of scraped samples that will be accepted.
  # If more than this number of samples are present after metric relabeling
  # the entire scrape will be treated as failed. 0 means no limit.
  [ sample_limit: <int> | default = 0 ]

  # Limit on the number of labels that will be accepted per sample. If more
  # than this number of labels are present on any sample post metric-relabeling,
  # the entire scrape will be treated as failed. 0 means no limit.
  [ label_limit: <int> | default = 0 ]

  # Limit on the length (in bytes) of each individual label name. If any label
  # name in a scrape is longer than this number post metric-relabeling, the
  # entire scrape will be treated as failed. Note that label names are UTF-8
  # encoded, and characters can take up to 4 bytes. 0 means no limit.
  [ label_name_length_limit: <int> | default = 0 ]

  # Limit on the length (in bytes) of each individual label value. If any label
  # value in a scrape is longer than this number post metric-relabeling, the
  # entire scrape will be treated as failed. Note that label values are UTF-8
  # encoded, and characters can take up to 4 bytes. 0 means no limit.
  [ label_value_length_limit: <int> | default = 0 ]

  # Limit per scrape config on number of unique targets that will be
  # accepted. If more than this number of targets are present after target
  # relabeling, Prometheus will mark the targets as failed without scraping them.
  # 0 means no limit. This is an experimental feature, this behaviour could
  # change in the future.
  [ target_limit: <int> | default = 0 ]

  # Limit per scrape config on the number of targets dropped by relabeling
  # that will be kept in memory. 0 means no limit.
  [ keep_dropped_targets: <int> | default = 0 ]

  # Specifies the validation scheme for metric and label names. Either blank or
  # "utf8" for full UTF-8 support, or "legacy" for letters, numbers, colons,
  # and underscores.
  [ metric_name_validation_scheme <string> | default "utf8" ]

runtime:
  # Configure the Go garbage collector GOGC parameter
  # See: https://tip.golang.org/doc/gc-guide#GOGC
  # Lowering this number increases CPU usage.
  [ gogc: <int> | default = 75 ]

# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files.
rule_files:
  [ - <filepath_glob> ... ]

# Scrape config files specifies a list of globs. Scrape configs are read from
# all matching files and appended to the list of scrape configs.
scrape_config_files:
  [ - <filepath_glob> ... ]

# A list of scrape configurations.
scrape_configs:
  [ - <scrape_config> ... ]

# Alerting specifies settings related to the Alertmanager.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# Settings related to the remote write feature.
remote_write:
  [ - <remote_write> ... ]

# Settings related to the OTLP receiver feature.
# See https://prometheus.io/docs/guides/opentelemetry/ for best practices.
otlp:
  [ promote_resource_attributes: [<string>, ...] | default = [ ] ]
  # Configures translation of OTLP metrics when received through the OTLP metrics
  # endpoint. Available values:
  # - "UnderscoreEscapingWithSuffixes" refers to commonly agreed normalization used
  #   by OpenTelemetry in https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/translator/prometheus
  # - "NoUTF8EscapingWithSuffixes" is a mode that relies on UTF-8 support in Prometheus.
  #   It preserves all special characters like dots, but still adds required metric name suffixes
  #   for units and _total, as UnderscoreEscapingWithSuffixes does.
  [ translation_strategy: <string> | default = "UnderscoreEscapingWithSuffixes" ]
  # Enables adding "service.name", "service.namespace" and "service.instance.id"
  # resource attributes to the "target_info" metric, on top of converting
  # them into the "instance" and "job" labels.
  [ keep_identifying_resource_attributes: <boolean> | default = false]

# Settings related to the remote read feature.
remote_read:
  [ - <remote_read> ... ]

# Storage related settings that are runtime reloadable.
storage:
  [ tsdb: <tsdb> ]
  [ exemplars: <exemplars> ]

# Configures exporting traces.
tracing:
  [ <tracing_config> ]
```

#### global

全局配置

#### scrape_config

指定需要抓取的目标集合以及抓取方式. 

抓取目标可以通过 `static_configs` 参数静态配置，也可使用支持的服务发现机制动态发现. 

可以通过 `relabel_config` 实现抓取前的对于任意目标及其标签的高级修改功能

#### config_relabels(重要)

标签重标记是非常强大的工具，可以在抓取目标之前动态重写抓取目标的标签集合. 标签的应用顺序与其在配置文件内的定义顺序相同. 

**元标签和 relabels_config 标签：** Labels starting with `__` will be removed from the label set after target relabeling is completed.

- 在已配置的标签之外，目标的会根据抓取配置自动生成一个 `job` 标签(默认设置为 job_name)和 `instance` 标签(默认设置为 `__address__`)，`__address__` 标签被设置为目标的 `<host>:<port>`
- `__meta_` 开头的标签是供重标记阶段使用的元标签，这些标签是抓取目标所提供的，由服务发现机制设置. 

```yaml
# The source_labels tells the rule what labels to fetch from the series. Any 
# labels which do not exist get a blank value ("").  Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <int> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```

`<regex>` is any valid [RE2 regular expression](https://github.com/google/re2/wiki/Syntax). It is required for the `replace`, `keep`, `drop`, `labelmap`,`labeldrop` and `labelkeep` actions. The regex is anchored on both ends. To un-anchor the regex, use `.*<regex>.*`.

`<relabel_action>` determines the relabeling action to take:

- `replace`: Match `regex` against the concatenated `source_labels`. Then, set `target_label` to `replacement`, with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value. If `regex` does not match, no replacement takes place.
- `lowercase`: Maps the concatenated `source_labels` to their lower case.
- `uppercase`: Maps the concatenated `source_labels` to their upper case.
- `keep`: Drop targets for which `regex` does not match the concatenated `source_labels`.
- `drop`: Drop targets for which `regex` matches the concatenated `source_labels`.
- `keepequal`: Drop targets for which the concatenated `source_labels` do not match `target_label`.
- `dropequal`: Drop targets for which the concatenated `source_labels` do match `target_label`.
- `hashmod`: Set `target_label` to the `modulus` of a hash of the concatenated `source_labels`.
- `labelmap`: Match `regex` against all source label names, not just those specified in `source_labels`. Then copy the values of the matching labels to label names given by `replacement` with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value.
- `labeldrop`: Match `regex` against all label names. Any label that matches will be removed from the set of labels.
- `labelkeep`: Match `regex` against all label names. Any label that does not match will be removed from the set of labels.

Care must be taken with `labeldrop` and `labelkeep` to ensure that metrics are still uniquely labeled once the labels are removed.

#### file_sd_config

读取指定文件中包含的 static_configs 配置属性. 将会通过磁盘监控机制对指定的文件动态监测并应用文件更改，以更新抓取配置. 

此机制不仅会监测这些单独的 static_configs 配置文件，文件的上级目录也默认被监视是否存在改动，这是为了检测是否存在符合配置的新配置文件.

#### dns_sd_config

基于 DNS 的服务发现机制，指定一组 DNS 域名以向其中周期性查询要抓取的目标列表. 

#### kubernetes_sd_config

从 k8s REST API 中获取抓取目标并与集群状态同步. 可配置的发现类型有如下：

- node

  对每个集群发现一个默认为 kubelet HTTP 端口的目标. 

- service

  对每个服务的端口发现一个目标，通常被用于服务的黑盒监控. 

- pod

  发现所有 pod 并暴露其中的容器作为目标

- endpoints

  发现服务的端点作为目标. 

- endpointslice

  端点切片目标

- ingress

  ingress目标

### Recording  Rules

记录规则允许你提前计算经常需要的或计算消耗大的表达式，并将其结果保存为新的一组时间序列, 以可以降低查询时需要的资源消耗. 这一点对于仪表盘来说尤其有用，因为其需要经常查询相同表达式的时间序列. 

记录规则或报警规则存在于规则组中，一个规则组中的规则会以固定的时间间隔按序运行，他们有相同的计算时间. 

规则的名称必须为有效的指标名称，警告规则的名称必须为有效的标签值

```yaml
# 规则文件的格式
groups:
  [ - <rule_group> ]

# <rule-group>
# The name of the group. Must be unique within a file.
name: <string>

# How often rules in the group are evaluated.
[ interval: <duration> | default = global.evaluation_interval ]

# Limit the number of alerts an alerting rule and series a recording
# rule can produce. 0 is no limit.
[ limit: <int> | default = 0 ]

# Offset the rule evaluation timestamp of this particular group by the specified duration into the past.
[ query_offset: <duration> | default = global.rule_query_offset ]

# Labels to add or overwrite before storing the result for its rules.
# Labels defined in <rule> will override the key if it has a collision.
labels:
  [ <labelname>: <labelvalue> ]

rules:
  [ - <rule> ... ]
  
# <rule>
# The name of the time series to output to. Must be a valid metric name.
record: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and the result recorded as a new set of
# time series with the metric name as given by 'record'.
expr: <string>

# Labels to add or overwrite before storing the result.
labels:
  [ <labelname>: <labelvalue> ]
```

```yaml
# 示例
groups:
  - name: example
    rules:
    - record: code:prometheus_http_requests_total:sum
      expr: sum by (code) (prometheus_http_requests_total)
```

### Alerting Rules

警告规则允许你基于 Prometheus 表达式来定义警报条件，并且向外部服务发送警报消息. 每当警报表达式计算出一个或多个向量元素时，这些元素的标签的警告计数就会激活. 

```yaml
# syntax for alerting rules
# The name of the alert. Must be a valid label value.
alert: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and all resultant time series become
# pending/firing alerts.
expr: <string>

# Alerts are considered firing once they have been returned for this long.
# Alerts which have not yet fired for long enough are considered pending.
[ for: <duration> | default = 0s ]

# How long an alert will continue firing after the condition that triggered it
# has cleared.
[ keep_firing_for: <duration> | default = 0s ]

# Labels to add or overwrite for each alert.
labels:
  [ <labelname>: <tmpl_string> ]

# Annotations to add to each alert.
annotations:
  [ <labelname>: <tmpl_string> ]
```

#### 定义警报规则

```yaml
groups:
- name: example
  labels:
    team: myteam
  rules:
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m # 
    keep_firing_for: 5m
    labels:
      severity: page
    annotations:
      summary: High request latency
```

#### 插值

```yaml
# To insert a firing element's label values:
{{ $labels.<labelname> }}
# To insert the numeric expression value of the firing element:
{{ $value }}
```

##  kube-prometheus 部署配置分析

### 1. prometheus-server 部署配置

#### 1.1 depolyment.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    prometheus-operator-input-hash: "11063300221446468566"
  creationTimestamp: "2025-05-07T09:16:18Z"
  generation: 1
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.54.1
    managed-by: prometheus-operator
    operator.prometheus.io/mode: server
    operator.prometheus.io/name: k8s
    operator.prometheus.io/shard: "0"
  name: prometheus-k8s
  namespace: monitoring
  ownerReferences:
  - apiVersion: monitoring.coreos.com/v1
    blockOwnerDeletion: true
    controller: true
    kind: Prometheus
    name: k8s
    uid: 3f516e01-c5f5-4412-95fc-41896a3c5833
  resourceVersion: "3514"
  uid: b555f120-1b47-4a39-9c59-21b88490a2df
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  podManagementPolicy: Parallel
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/managed-by: prometheus-operator
      app.kubernetes.io/name: prometheus
      operator.prometheus.io/name: k8s
      operator.prometheus.io/shard: "0"
      prometheus: k8s
  serviceName: prometheus-operated
  template:
    metadata:
      annotations:
        kubectl.kubernetes.io/default-container: prometheus
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: prometheus
        app.kubernetes.io/instance: k8s
        app.kubernetes.io/managed-by: prometheus-operator
        app.kubernetes.io/name: prometheus
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 2.54.1
        operator.prometheus.io/name: k8s
        operator.prometheus.io/shard: "0"
        prometheus: k8s
    spec:
      automountServiceAccountToken: true
      containers:
      - args:
        - --web.console.templates=/etc/prometheus/consoles
        - --web.console.libraries=/etc/prometheus/console_libraries
        - --config.file=/etc/prometheus/config_out/prometheus.env.yaml
        - --web.enable-lifecycle
        - --web.route-prefix=/
        - --storage.tsdb.retention.time=24h
        - --storage.tsdb.path=/prometheus
        - --web.config.file=/etc/prometheus/web_config/web-config.yaml
        image: quay.io/prometheus/prometheus:v2.54.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 6
          httpGet:
            path: /-/healthy
            port: web
            scheme: HTTP
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 3
        name: prometheus
        ports:
        - containerPort: 9090
          name: web
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /-/ready
            port: web
            scheme: HTTP
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 3
        resources:
          requests:
            memory: 400Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        startupProbe:
          failureThreshold: 60
          httpGet:
            path: /-/ready
            port: web
            scheme: HTTP
          periodSeconds: 15
          successThreshold: 1
          timeoutSeconds: 3
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        volumeMounts:
        - mountPath: /etc/prometheus/config_out
          name: config-out
          readOnly: true
        - mountPath: /etc/prometheus/certs
          name: tls-assets
          readOnly: true
        - mountPath: /prometheus
          name: prometheus-k8s-db
        - mountPath: /etc/prometheus/rules/prometheus-k8s-rulefiles-0
          name: prometheus-k8s-rulefiles-0
        - mountPath: /etc/prometheus/web_config/web-config.yaml
          name: web-config
          readOnly: true
          subPath: web-config.yaml
      - args:
        - --listen-address=:8080
        - --reload-url=http://localhost:9090/-/reload
        - --config-file=/etc/prometheus/config/prometheus.yaml.gz
        - --config-envsubst-file=/etc/prometheus/config_out/prometheus.env.yaml
        - --watched-dir=/etc/prometheus/rules/prometheus-k8s-rulefiles-0
        command:
        - /bin/prometheus-config-reloader
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: SHARD
          value: "0"
        image: quay.io/prometheus-operator/prometheus-config-reloader:v0.76.2
        imagePullPolicy: IfNotPresent
        name: config-reloader
        ports:
        - containerPort: 8080
          name: reloader-web
          protocol: TCP
        resources:
          limits:
            cpu: 10m
            memory: 50Mi
          requests:
            cpu: 10m
            memory: 50Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        volumeMounts:
        - mountPath: /etc/prometheus/config
          name: config
        - mountPath: /etc/prometheus/config_out
          name: config-out
        - mountPath: /etc/prometheus/rules/prometheus-k8s-rulefiles-0
          name: prometheus-k8s-rulefiles-0
      dnsPolicy: ClusterFirst
      initContainers:
      - args:
        - --watch-interval=0
        - --listen-address=:8081
        - --config-file=/etc/prometheus/config/prometheus.yaml.gz
        - --config-envsubst-file=/etc/prometheus/config_out/prometheus.env.yaml
        - --watched-dir=/etc/prometheus/rules/prometheus-k8s-rulefiles-0
        command:
        - /bin/prometheus-config-reloader
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: SHARD
          value: "0"
        image: quay.io/prometheus-operator/prometheus-config-reloader:v0.76.2
        imagePullPolicy: IfNotPresent
        name: init-config-reloader
        ports:
        - containerPort: 8081
          name: reloader-web
          protocol: TCP
        resources:
          limits:
            cpu: 10m
            memory: 50Mi
          requests:
            cpu: 10m
            memory: 50Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        volumeMounts:
        - mountPath: /etc/prometheus/config
          name: config
        - mountPath: /etc/prometheus/config_out
          name: config-out
        - mountPath: /etc/prometheus/rules/prometheus-k8s-rulefiles-0
          name: prometheus-k8s-rulefiles-0
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 1000
      serviceAccount: prometheus-k8s
      serviceAccountName: prometheus-k8s
      shareProcessNamespace: false
      terminationGracePeriodSeconds: 600
      volumes:
      - name: config
        secret:
          defaultMode: 420
          secretName: prometheus-k8s
      - name: tls-assets
        projected:
          defaultMode: 420
          sources:
          - secret:
              name: prometheus-k8s-tls-assets-0
      - emptyDir:
          medium: Memory
        name: config-out
      - configMap:
          defaultMode: 420
          name: prometheus-k8s-rulefiles-0
        name: prometheus-k8s-rulefiles-0
      - name: web-config
        secret:
          defaultMode: 420
          secretName: prometheus-k8s-web-config
      - emptyDir: {}
        name: prometheus-k8s-db
  updateStrategy:
    type: RollingUpdate
status:
  availableReplicas: 2
  collisionCount: 0
  currentReplicas: 2
  currentRevision: prometheus-k8s-5dc5d5697
  observedGeneration: 1
  readyReplicas: 2
  replicas: 2
  updateRevision: prometheus-k8s-5dc5d5697
  updatedReplicas: 2
```

#### 1.2 prometheus-server 相关的其他 yaml 文件

```shell
# 服务账号、集群权限及+角色绑定相关配置文件
prometheus-serviceAccount.yaml
prometheus-clusterRole.yaml
prometheus-clusterRoleBinding.yaml
prometheus-roleConfig.yaml
prometheus-roleBindingConfig.yaml
prometheus-roleSpecificNamespaces.yaml
prometheus-roleBindingSpecificNamespaces.yaml
# prometheus-server 部署配置文件
prometheus-prometheus.yaml
# prometheus 相关报警规则配置文件
prometheus-prometheusRule.yaml
# prometheus 相关服务监视配置文件
prometheus-serviceMonitor.yaml
# 
prometheus-service.yaml
# prometheus 相关服务网络流量路由策略文件
prometheus-networkPolicy.yaml
# prometheus 相关 pod 可用性声明文件
# an object to define the max disruption that can be caused to a collection of pods
prometheus-podDisruptionBudget.yaml
```


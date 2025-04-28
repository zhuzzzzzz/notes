### 基于 Helm 部署 redis 主从集群

#### 1. 安装 helm

```shell
# 脚本安装(Linux)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# 二进制安装
wget https://get.helm.sh/helm-v3.10.0-linux-amd64.tar.gz
tar -zxvf helm-v3.10.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/

# apt方式安装
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

#### 2. helm 基础操作

```shell
# 查看默认仓库
helm repo list

# 添加仓库
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add bitnami https://helm-charts.itboon.top/bitnami 
# 更新仓库
helm repo update
# 删除仓库
helm repo remove aliyun

# 搜索 hub 内的chart
helm search hub redis
# 搜索仓库内的 redis chart
helm search repo redis

# 查看安装说明
helm show readme bitnami/redis

# 查看 chart 中的可配置选项
helm show values bitnami/redis

# 使用 yaml 格式的文件覆盖默认的配置项进行安装
helm install -f values.yaml xxx bitnami/redis
helm install -f values.yaml bitnami/redis --generate-name

# 更多安装方法
# 本地 chart 压缩包
helm install foo foo-0.1.1.tgz
# 解压后的 chart 目录
helm install foo path/to/foo
# 完整的 url
helm install foo https://example.com/charts/foo-1.2.3.tgz

# 查看 release 状态
helm status release-name

# 升级
helm upgrade -f panda.yaml happy-panda bitnami/wordpress

# 查看历史版本
helm history happy-panda

# 回滚
helm rollback happy-panda 1

# 查看 release 的配置信息
helm get values
```

#### 3. 配置 StroageClass

##### 3.1  local-path-provisioner

参考[文档](https://gitcode.com/gh_mirrors/lo/local-path-provisioner?utm_source=highlight_word_gitcode&word=local-path-provisioner). 

```shell
# 获取 sc 定义文件
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml

# 修改 sc 镜像配置, 创建 sc
image: m.daocloud.io/docker.io/rancher/local-path-provisioner:v0.0.28
image: m.daocloud.io/docker.io/library/busybox
```

#### 4. 安装 redis chart

```shell
# 将 chart 拉取到本地
helm pull bitnami/redis

# 解压
tar -xf redis.tgz

# 配置 values.yaml 
imageRegistry: m.daocloud.io/docker.io
storageClasss: (defaultStorageClass:) "local-path"
persistence:
  size: 1Gi
  
# 安装 redis
helm install redis .

# 移除 redis
helm uninstall redis 
```

#### 5. 安装成功

```shell
# Redis&reg; can be accessed on the following DNS names from within your cluster:
redis-master.default.svc.cluster.local for read/write operations (port 6379)
redis-replicas.default.svc.cluster.local for read-only operations (port 6379)

# To get your password run:
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)
```

**To connect to your Redis&reg; server:**

1. Run a Redis&reg; pod that you can use as a client:

   ```shell
   kubectl run --namespace default redis-client --restart='Never' --env REDIS_PASSWORD=$REDIS_PASSWORD --image m.daocloud.io/docker.io/bitnami/redis:7.4.2-debian-12-r6 --command -- sleep infinity
   ```

   Use the following command to attach to the pod:

   ```shell
   kubectl exec --tty -i redis-client \
   --namespace default -- bash
   ```

2. Connect using the Redis&reg; CLI:

   ```shell
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-replicas
   ```

**To connect to your database from outside the cluster execute the following commands:**

    kubectl port-forward --namespace default svc/redis-master 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379

#### 6. 升级与回滚

##### 6.1 升级

```shell
# 修改密码
vi values.yaml

# 执行更新操作( "." 指代当前 chart 目录)
helm upgrade redis .
```

##### 6.2 回滚

```shell
# 
helm history redis -n default

# 
helm rollback redis 1  
```

##### 6.3 管理

```shell
# 
helm list

# 
helm uninstall redis

# 
helm delete redis
```

##### 6.4 关于 pvc

默认不会删除 pvc ，可以手动删除以释放占用的磁盘资源

### 部署配置文件分析

#### 0. 使用 kubectl explain 查看字段说明

```shell
kubectl explain <type>.<fieldName>[.<fieldName>]
```

#### 1. Redis master YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    meta.helm.sh/release-name: redis
    meta.helm.sh/release-namespace: default
  generation: 1 # 自动生成，用于跟踪spec的变更历史
  labels:
    app.kubernetes.io/component: master
    app.kubernetes.io/instance: redis
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: redis
    app.kubernetes.io/version: 7.4.2
    helm.sh/chart: redis-20.11.4
  name: redis-master
  namespace: default
  resourceVersion: "691531" # 自动生成，表示资源的当前版本，每当对象的任何部分(包括spec和status)发生变化，更新该值，用于乐观锁和watch机制
  uid: 28bf9d7c-8b7b-4543-84ed-d342a39d6233 # 自动生成，资源的全局唯一标识符
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain # sts删除时，关联的pvc和底层存储卷的保留策略
    whenScaled: Retain # sts扩容缩容时，关联的pvc和底层存储卷的保留策略
  podManagementPolicy: OrderedReady # 或Parallel，两种策略
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: master
      app.kubernetes.io/instance: redis
      app.kubernetes.io/name: redis
  serviceName: redis-headless
  template:
    metadata:
      annotations:
        checksum/configmap: 2a9ab4a5432825504d910f022638674ce88eaefe9f9f595ad8bc107377d104fb
        checksum/health: aff24913d801436ea469d8d374b2ddb3ec4c43ee7ab24663d5f8ff1a1b6991a9
        checksum/scripts: 2402f1a299715d378c24e380602cbccb9b58b12e8d914394b3ba78ade9f4f16f
        checksum/secret: 3f70900d596125b197f330334264d3e2855b320a5123ffadc194e495f3688c4d
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: master
        app.kubernetes.io/instance: redis
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: redis
        app.kubernetes.io/version: 7.4.2
        helm.sh/chart: redis-20.11.4
    spec:
      affinity:
        # 确保一个节点只运行一个redis master
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/component: master
                  app.kubernetes.io/instance: redis
                  app.kubernetes.io/name: redis
              topologyKey: kubernetes.io/hostname
            weight: 1
      automountServiceAccountToken: false
      containers:
      - args:
        - -c
        - /opt/bitnami/scripts/start-scripts/start-master.sh
        command:
        - /bin/bash
        env:
        - name: BITNAMI_DEBUG
          value: "false"
        - name: REDIS_REPLICATION_MODE
          value: master
        - name: ALLOW_EMPTY_PASSWORD
          value: "no"
        - name: REDIS_PASSWORD_FILE
          value: /opt/bitnami/redis/secrets/redis-password
        - name: REDIS_TLS_ENABLED
          value: "no"
        - name: REDIS_PORT
          value: "6379"
        image: m.daocloud.io/docker.io/bitnami/redis:7.4.2-debian-12-r6
        imagePullPolicy: IfNotPresent
        livenessProbe: # 存活探针，失败后容器会被重启(根据restartPolicy)
          exec:
            command:
            - sh
            - -c
            - /health/ping_liveness_local.sh 5
          failureThreshold: 5 # 被认为失效的连续探测失败次数
          initialDelaySeconds: 20 # 容器启动后的检测时延
          periodSeconds: 5 # 探测间隔
          successThreshold: 1 # 被认为成功的连续探测成功次数
          timeoutSeconds: 6 # 超时时间
        name: redis
        ports:
        - containerPort: 6379
          name: redis
          protocol: TCP
        readinessProbe: # 就绪探针，失败后容器会被从Service的Endpoints中移除，停止接收流量，不会重启
          exec:
            command:
            - sh
            - -c
            - /health/ping_readiness_local.sh 1
          failureThreshold: 5
          initialDelaySeconds: 20
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 2
        resources:
          limits:
            cpu: 150m
            ephemeral-storage: 2Gi
            memory: 192Mi
          requests:
            cpu: 100m
            ephemeral-storage: 50Mi
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsGroup: 1001
          runAsNonRoot: true
          runAsUser: 1001
          seLinuxOptions: {}
          seccompProfile:
            type: RuntimeDefault
        terminationMessagePath: /dev/termination-log # 容器内挂载的终止信息文件存储路径
        terminationMessagePolicy: File # 如何生成终止信息，从文件读取；FallbackToLogsOnError，从容器日志读取最近的内容
        volumeMounts:
        - mountPath: /opt/bitnami/scripts/start-scripts
          name: start-scripts
        - mountPath: /health
          name: health
        - mountPath: /opt/bitnami/redis/secrets/
          name: redis-password
        - mountPath: /data
          name: redis-data
        - mountPath: /opt/bitnami/redis/mounted-etc
          name: config
        - mountPath: /opt/bitnami/redis/etc/
          name: empty-dir
          subPath: app-conf-dir
        - mountPath: /tmp
          name: empty-dir
          subPath: tmp-dir
      dnsPolicy: ClusterFirst # 
      enableServiceLinks: true # 将当前命名空间下的存在的服务资源信息添加到pod环境变量
      restartPolicy: Always # pod内容器的重启策略
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1001
        fsGroupChangePolicy: Always
      serviceAccount: redis-master
      serviceAccountName: redis-master
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap: # 使用configMap填充此卷
          defaultMode: 493 # 755权限
          name: redis-scripts
        name: start-scripts
        # start-master.sh, start-replica.sh
      - configMap:
          defaultMode: 493
          name: redis-health
        name: health
        # ping_liveness_local.sh, ping_readiness_local.sh  master节点执行的探针命令
        # ping_liveness_local_and_master.sh, ping_readiness_local_and_master.sh  replica节点执行的探针命令，调用上下两种脚本
        # ping_liveness_master.sh,  ping_readiness_master.sh 
      - name: redis-password
        secret: # 使用secret填充此卷
          defaultMode: 420 # 644权限
          items:
          - key: redis-password
            path: redis-password
          secretName: redis
      - configMap:
          defaultMode: 420
          name: redis-configuration
        name: config
      - emptyDir: {}
        name: empty-dir
  updateStrategy:
    type: RollingUpdate # OnDelete, 只有手动删除时才会更新pod；RollingUpdate，滚动更新
  volumeClaimTemplates: # PVC列表
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: master
        app.kubernetes.io/instance: redis
        app.kubernetes.io/name: redis
      name: redis-data
    spec:
      accessModes:
      - ReadWriteOnce # 只能由单个pod读写，同节点的其他pod可读
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-path
      volueMode: Filesystem # Block或Filesystem
    status:
      phase: Pending
status:
  availableReplicas: 1
  collisionCount: 0
  currentReplicas: 1
  currentRevision: redis-master-5b54b4bcdd
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updateRevision: redis-master-5b54b4bcdd
  updatedReplicas: 1

```

#### 2. Redis replica YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    meta.helm.sh/release-name: redis
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-04-28T06:47:53Z"
  generation: 1
  labels:
    app.kubernetes.io/component: replica
    app.kubernetes.io/instance: redis
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: redis
    app.kubernetes.io/version: 7.4.2
    helm.sh/chart: redis-20.11.4
  name: redis-replicas
  namespace: default
  resourceVersion: "13350"
  uid: 986a0b05-fc80-4ca7-a22c-81f518d2eaea
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: replica
      app.kubernetes.io/instance: redis
      app.kubernetes.io/name: redis
  serviceName: redis-headless
  template:
    metadata:
      annotations:
        checksum/configmap: 2a9ab4a5432825504d910f022638674ce88eaefe9f9f595ad8bc107377d104fb
        checksum/health: aff24913d801436ea469d8d374b2ddb3ec4c43ee7ab24663d5f8ff1a1b6991a9
        checksum/scripts: 2402f1a299715d378c24e380602cbccb9b58b12e8d914394b3ba78ade9f4f16f
        checksum/secret: 03ed3938b77c7727208a32ade0193c1363d371f217c14b383fa460d4b9a98780
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: replica
        app.kubernetes.io/instance: redis
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: redis
        app.kubernetes.io/version: 7.4.2
        helm.sh/chart: redis-20.11.4
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/component: replica
                  app.kubernetes.io/instance: redis
                  app.kubernetes.io/name: redis
              topologyKey: kubernetes.io/hostname
            weight: 1
      automountServiceAccountToken: false
      containers:
      - args:
        - -c
        - /opt/bitnami/scripts/start-scripts/start-replica.sh
        command:
        - /bin/bash
        env:
        - name: BITNAMI_DEBUG
          value: "false"
        - name: REDIS_REPLICATION_MODE
          value: replica
        - name: REDIS_MASTER_HOST
          value: redis-master-0.redis-headless.default.svc.cluster.local
        - name: REDIS_MASTER_PORT_NUMBER
          value: "6379"
        - name: ALLOW_EMPTY_PASSWORD
          value: "no"
        - name: REDIS_PASSWORD_FILE
          value: /opt/bitnami/redis/secrets/redis-password
        - name: REDIS_MASTER_PASSWORD_FILE
          value: /opt/bitnami/redis/secrets/redis-password
        - name: REDIS_TLS_ENABLED
          value: "no"
        - name: REDIS_PORT
          value: "6379"
        image: m.daocloud.io/docker.io/bitnami/redis:7.4.2-debian-12-r6
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - /health/ping_liveness_local_and_master.sh 5
          failureThreshold: 5
          initialDelaySeconds: 20
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 6
        name: redis
        ports:
        - containerPort: 6379
          name: redis
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - /health/ping_readiness_local_and_master.sh 1
          failureThreshold: 5
          initialDelaySeconds: 20
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 2
        resources:
          limits:
            cpu: 150m
            ephemeral-storage: 2Gi
            memory: 192Mi
          requests:
            cpu: 100m
            ephemeral-storage: 50Mi
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsGroup: 1001
          runAsNonRoot: true
          runAsUser: 1001
          seLinuxOptions: {}
          seccompProfile:
            type: RuntimeDefault
        startupProbe:
          failureThreshold: 22
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: redis
          timeoutSeconds: 5
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt/bitnami/scripts/start-scripts
          name: start-scripts
        - mountPath: /health
          name: health
        - mountPath: /opt/bitnami/redis/secrets/
          name: redis-password
        - mountPath: /data
          name: redis-data
        - mountPath: /opt/bitnami/redis/mounted-etc
          name: config
        - mountPath: /opt/bitnami/redis/etc
          name: empty-dir
          subPath: app-conf-dir
        - mountPath: /tmp
          name: empty-dir
          subPath: tmp-dir
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1001
        fsGroupChangePolicy: Always
      serviceAccount: redis-replica
      serviceAccountName: redis-replica
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 493
          name: redis-scripts
        name: start-scripts
      - configMap:
          defaultMode: 493
          name: redis-health
        name: health
      - name: redis-password
        secret:
          defaultMode: 420
          items:
          - key: redis-password
            path: redis-password
          secretName: redis
      - configMap:
          defaultMode: 420
          name: redis-configuration
        name: config
      - emptyDir: {}
        name: empty-dir
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: replica
        app.kubernetes.io/instance: redis
        app.kubernetes.io/name: redis
      name: redis-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-path
      volumeMode: Filesystem
    status:
      phase: Pending
status:
  availableReplicas: 3
  collisionCount: 0
  currentReplicas: 3
  currentRevision: redis-replicas-5447cc9d45
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updateRevision: redis-replicas-5447cc9d45
  updatedReplicas: 3
```

#### 3.  启动脚本

##### 1. master 启动脚本

```shell
#!/bin/bash

[[ -f $REDIS_PASSWORD_FILE ]] && export REDIS_PASSWORD="$(< "${REDIS_PASSWORD_FILE}")"
if [[ -f /opt/bitnami/redis/mounted-etc/master.conf ]];then
    cp /opt/bitnami/redis/mounted-etc/master.conf /opt/bitnami/redis/etc/master.conf
fi
if [[ -f /opt/bitnami/redis/mounted-etc/redis.conf ]];then
    cp /opt/bitnami/redis/mounted-etc/redis.conf /opt/bitnami/redis/etc/redis.conf
fi
if [[ -f /opt/bitnami/redis/mounted-etc/users.acl ]];then
    cp /opt/bitnami/redis/mounted-etc/users.acl /opt/bitnami/redis/etc/users.acl
fi
ARGS=("--port" "${REDIS_PORT}")
ARGS+=("--requirepass" "${REDIS_PASSWORD}")
ARGS+=("--masterauth" "${REDIS_PASSWORD}")
ARGS+=("--include" "/opt/bitnami/redis/etc/redis.conf")
ARGS+=("--include" "/opt/bitnami/redis/etc/master.conf")
exec redis-server "${ARGS[@]}"
```

##### 2. replica 启动脚本

```shell
#!/bin/bash

get_port() {
    hostname="$1"
    type="$2"

    port_var=$(echo "${hostname^^}_SERVICE_PORT_$type" | sed "s/-/_/g")
    port=${!port_var}

    if [ -z "$port" ]; then
        case $type in
            "SENTINEL")
                echo 26379
                ;;
            "REDIS")
                echo 6379
                ;;
        esac
    else
        echo $port
    fi
}

get_full_hostname() {
    hostname="$1"
    full_hostname="${hostname}.${HEADLESS_SERVICE}"
    echo "${full_hostname}"
}

REDISPORT=$(get_port "$HOSTNAME" "REDIS")
HEADLESS_SERVICE="redis-headless.default.svc.cluster.local"

[[ -f $REDIS_PASSWORD_FILE ]] && export REDIS_PASSWORD="$(< "${REDIS_PASSWORD_FILE}")"
[[ -f $REDIS_MASTER_PASSWORD_FILE ]] && export REDIS_MASTER_PASSWORD="$(< "${REDIS_MASTER_PASSWORD_FILE}")"
if [[ -f /opt/bitnami/redis/mounted-etc/replica.conf ]];then
    cp /opt/bitnami/redis/mounted-etc/replica.conf /opt/bitnami/redis/etc/replica.conf
fi
if [[ -f /opt/bitnami/redis/mounted-etc/redis.conf ]];then
    cp /opt/bitnami/redis/mounted-etc/redis.conf /opt/bitnami/redis/etc/redis.conf
fi
if [[ -f /opt/bitnami/redis/mounted-etc/users.acl ]];then
    cp /opt/bitnami/redis/mounted-etc/users.acl /opt/bitnami/redis/etc/users.acl
fi

echo "" >> /opt/bitnami/redis/etc/replica.conf
echo "replica-announce-port $REDISPORT" >> /opt/bitnami/redis/etc/replica.conf
echo "replica-announce-ip $(get_full_hostname "$HOSTNAME")" >> /opt/bitnami/redis/etc/replica.conf
ARGS=("--port" "${REDIS_PORT}")
ARGS+=("--replicaof" "${REDIS_MASTER_HOST}" "${REDIS_MASTER_PORT_NUMBER}")
ARGS+=("--requirepass" "${REDIS_PASSWORD}")
ARGS+=("--masterauth" "${REDIS_MASTER_PASSWORD}")
ARGS+=("--include" "/opt/bitnami/redis/etc/redis.conf")
ARGS+=("--include" "/opt/bitnami/redis/etc/replica.conf")
exec redis-server "${ARGS[@]}"
```


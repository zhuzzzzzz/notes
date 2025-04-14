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
```

#### 2. 搜索配置 helm 仓库

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

# 搜索 redis chart
helm search repo redis

# 查看安装说明
helm show readme bitnami/redis
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

**Redis&reg; can be accessed on the following DNS names from within your cluster:**

    redis-master.default.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.default.svc.cluster.local for read-only operations (port 6379)

**To get your password run:**

    export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}"     | base64 -d)

**To connect to your Redis&reg; server:**

1. Run a Redis&reg; pod that you can use as a client:

   ```shell
   kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --im    age m.daocloud.io/docker.io/bitnami/redis:7.4.2-debian-12-r6 --command -- sleep infinity
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
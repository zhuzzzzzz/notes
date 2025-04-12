### 一、 安装 helm

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

### 二、搜索配置 helm 仓库

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
helm show readme aliyun/redis
```

### 三、 安装 redis chart

```shell
# 将 chart 拉取到本地
helm pull aliyun/redis-ha

# 解压
tar -xf redis-ha.tgz
```


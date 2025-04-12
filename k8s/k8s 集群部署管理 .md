# 部署 kubenetes 集群 (ubuntu)

## 一、安装 kubeadm

 [kubenetes 文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### 1. 准备工作

- 兼容的 Linux 主机

- 2 GB 以上内存

- 2个以上 CPU

- 所有集群机器之间的网络连通

- 集群每个节点应具有不同的主机名、MAC 地址以及 product_uuid

- 开放端口 6443

### 2. 核实所有节点的 MAC 地址及 product_uuid 的唯一性

```shell
ifconfig -a
```

```shell
sudo cat /sys/class/dmi/id/product_uuid
```

### 3. 关闭所有节点的 swap 配置

- tolerate swap

- /etc/fstab 中注释掉 swap 行以禁用 swap 分区(重启后生效)

### 4. 为每个节点安装并配置容器运行时

#### 4.1 containerd

[containerd 文档](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)

containerd 使用的 CRI 套接字为 /run/containerd/containerd.sock

##### 4.1.1 安装

- 安装 docker-ce 时将自动安装 containerd, 而 containerd.io 可单独安装

##### 4.1.2 配置

- 生成默认配置文件

  ```shell
  containerd config default > /etc/containerd/config.toml
  ```

- containerd 需启用 CRI 支持，确保 cri 不在配置文件 /etc/containerd/config.toml 的 disabled_plugins 列表中

- 修改配置文件，配置 runc 使用 systemd cgroup driver

  ```toml
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    ...
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
  ```

- 修改配置文件，配置 sandbox image 以使用可用的镜像源

  ```toml
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.k8s.io/pause:3.2"
  ```

- 重启 containerd

  ```shell
  sudo systemctl restart containerd
  ```

#### 4.2 CRI-O

#### 4.3 Docker Engine(cri-dockerd)

### 5. 安装 kubeadm kubelet kubectl

[安装文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

kubeadm 安装的 k8s 控制平面版本需要与 kubelet 和 kubectl 保持兼容。一般来说，kubelet 与 控制平面的一个小版本号差异是允许的，但 kubelet 的版本不应该超过 API server 的版本。例如 1.7.0 的 kubelet 与 1.8.0 的 API server 是兼容的，但反之不一定成立。

#### 5.1 For Debian-based(ubuntu)

- 更新 apt 索引并安装依赖的包
  
  ```shell
  sudo apt-get update
  # apt-transport-https may be a dummy package; if so, you can skip that package
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg
  ```

- k8s 公共签名密钥
  
  ```shell
  # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
  # sudo mkdir -p -m 755 /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  ```

- 添加 k8s 仓库源
  
  ```shell
  # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```

- 安装
  
  ```shell
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
  ```

- 可选操作，执行后 kebulet 会每个几秒重启一次，等待 kubeadm 的指令

  ```shell
  sudo systemctl enable --now kubelet
  ```

#### 5.2 For Red Hat-based(centos)

### 6. 配置 cgroup driver

容器运行时以及 kubelet 的 cgroup driver 设置需要匹配，应都配置为 systemd

注：

In v1.22 and later, if the user does not set the `cgroupDriver` field under `KubeletConfiguration`, kubeadm defaults it to `systemd`.

In Kubernetes v1.28, you can enable automatic detection of the cgroup driver as an alpha feature. See [systemd cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#systemd-cgroup-driver) for more details.

## 二、使用 kubeadm 创建集群

### 1. 初始化控制平面节点

执行指令初始化控制平面节点

```shell
kubeadm init <args>
```

#### ```--control-plane-endpoint```

当未设置```control-plane-endpoint```时，创建的单节点控制平面无法在后续的配置中修改成为高可用性的集群。因此若要创建高可用性集群，在初始化时应设置此项，并在后续的配置中修改```cluster-point```指向负载均衡服务器的地址

#### ```--pod-network-cidr```

根据选用的 Pod 网络插件，核实是否需要在 kubeadm init 时传递其他参数。根据第三方插件的不同，可能需要设置```-pod-network-cidr```

#### ```--cri-socket```

kubeadm 会尝试检测容器运行时，但当节点提供多个运行时时，需要指出使用哪一个容器运行时

#### ```--apiserver-advertise-address```

```--apiserver-advertise-address``` 仅能设置当前节点的 API 服务地址，而 ```--control-plane-endpoint```  可设置整个控制平面节点的共享端口

#### ```--upload-certs```

设置此项将上传所有在集群控制平面中共享的证书。如果希望在控制平面节点之间手动复制证书则不要设置此项。

#### ```config```

导出默认的配置文件以供修改。设置此项与 ```--certificate-key``` 冲突，若使用配置文件的方式初始化集群，需手动在 ```InitConfiguration``` 及 ```JoinConfiguration: controlPlane``` 中添加 ```certificateKey``` 字段。

```shell
kubeadm config print init-defaults > kubeadm-config.yaml
```

参考测试环境用到的配置文件如下

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.2.223
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  imagePullSerial: true
  taints: null
---
apiVersion: kubeadm.k8s.io/v1beta4
controlPlaneEndpoint: "172.16.2.223:6443"
clusterName: kubernetes
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.31.0
networking:
  podSubnet: 10.244.0.0/16
```

#### ```--config```

```shell
kubeadm init --config  kubeadm-config.yaml --upload-certs  # 执行命令初始化 k8s 控制平面
```

![image-20250313171320090](C:\Users\zhu\AppData\Roaming\Typora\typora-user-images\image-20250313171320090.png)

![image-20250313171339512](C:\Users\zhu\AppData\Roaming\Typora\typora-user-images\image-20250313171339512.png)

### 2. 安装 Pod 网络插件

需要部署一个基于 CNI 的 Pod 网络插件，这样 Pod 之间才可以互相通信，集群的 DNS(CoreDNS) 服务将在网络插件安装后启动

#### 2.1 Flannel

- 应用配置文件部署 Flannel，使用 Flannel 时应设置 ```--pod-network-cidr```与 Flannel 配置 ```podCIDR``` 相同

  ```shell
  kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
  ```

- 若需要自定义配置，如 ```podCIDR``` (非 ```10.244.0.0/16```)，将文件下载至本地并做相应修改，如修改 flannel 所使用的镜像源

- 插件存在启动错误时可能需要重启 containerd

#### 2.2 calico



### 3. 添加后续节点

#### 3.1 准备工作

- 列出所有 token

  ```shell
  kubeadm token list
  ```

- 打印节点的加入指令

  ```shell
  kubeadm token create --print-join-command
  ```

- 当证书密钥过期

  ```shell
  kubeadm init phase upload-certs --upload-certs
  ```

- 生成完成的节点加入指令(管理节点)

  ```shell
  kubeadm token create --print-join-command --certificate-key 9e53a89876975e72a0f56271dc5bc9b0f57feda9c3ec855dd39b4af3f0dc16f7
  ```

#### 3.2 添加节点

使用创建集群时的输出指令将节点添加至集群

- 加入工作节点

  ```shell
  kubeadm join 172.16.2.85:6443 --token s0iayr.wahygdti66qk23v3 \
          --discovery-token-ca-cert-hash sha256:3b3d41568750887847db03e487eb8bdbd06e0700983cee7c75964da50e0fcd7c
  ```

- 加入管理节点

  ```shell
  kubeadm join 172.16.2.223:6443 --token 5gldx5.mbej61naf6em6u0e --discovery-token-ca-cert-hash sha256:7768a3835ad58151f066fe0c0f4936963f2e6278ede0bf7e0ab0e42e3a868967 --control-plane --certificate-key 9e53a89876975e72a0f56271dc5bc9b0f57feda9c3ec855dd39b4af3f0dc16f7
  ```


## 三、使用 kubectl 管理集群

### 3.1 常用管理命令

####  3.1.1 kubectl get

```shell
kubectl get resource -A  # 获取所有命名空间下的资源
kubectl get resource -n xxx  # 获取指定命名空间下的资源
kubectl get resource --namespace xxx  # 获取指定命名空间下的资源
kubectl get pod -A -o wide  # 获取pod并显示详细信息
kubectl get all  # 获取集群下所有资源
```

#### 3.1.2 kubectl create

```shell
kubectl create deployment nginx-deployment --image=nginx --replica=3
```

配置文件

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.openanolis.cn/openanolis/nginx:1.14.1-8.6
          ports:
            - containerPort: 80
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
```

```shell
kubectl create -f xxx.yaml  # 通过配置文件启动
kubectl apply -f xxx.yaml  # 通过配置文件创建或更新(不要求完整的配置文件)
```

#### 3.1.3 kubectl delete

```shell
kubectl delete resource xxx -n xxx
```

#### 3.1.4 kubectl logs

```shell
kubectl logs pod -c container -n xxx 
```

#### 3.1.5 kubectl exec

```shell
kubectl exec -it podname -- /bin/bash
```

#### 3.1.6 kubectl describe

```shell
# 查看 pod 情况
kubectl describe pod <pod-name> -n namespace
```

#### 3.1.7 kubectl label 

```shell
# 为 pod(或资源) 添加标签
kubectl label po pod app=hello -n kube-public
# 为 pod(或资源) 更改标签
kubectl label po pod app=hello2 --overwrite
# selector 按照 label 单值查找
kubectl get po -A -l app=hello
# 查看 lables
kubectl get po --show-labels
```

#### 3.1.8 验证 YAML 文件

```shell
kubectl apply --dry-run=client -f <file>  # 试运行检查语法错误。
kubectl explain <resource>  # 查看资源的合法字段及大小写格式。
```

### 3.2 部署 nginx 案例

#### 3.2.1 创建 nginx deployment

```shell
kubectl create -f nginx-deployment.yaml  # 使用配置文件部署nginx服务
curl ip-address  # 验证 nginx 的部署情况
```

#### 3.2.2 配置内部服务发现(Service)

##### 0. 支持的服务类型

**ClusterIP**		默认类型，用于集群内部访问

**NodePort**		节点端口类型，将服务公开到集群节点上(会在所有 kube-proxy 节点都绑定一个端口，此端口可以代理至对于 pod，集群外部可以使用任意节点 ip + NodePort 端口号 来访问集群中对应 pod )

**ExternalName**		外部名称类型，将服务映射到一个外部域名上

**LoadBalancer**		负载均衡类型，将服务公开到外部负载均衡服务器上

**Headless**		无头类型，主要用于DNS解析和服务发现

##### 1. ClusterIP

```shell
kubectl create service nginx-service
# 或
kubectl expose deployment nginx-deployment
# 操作完成后可以看到 service 已被创建成功
kubectl delete service nginx-deployment
```

使用配置文件创建 service

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```shell
kubectl apply -f nginx-service.yaml
```

##### 2. NodePort

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

##### 3. ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-external
  labels:
    app: nginx-svc
spec:
  type: ExternalName
  externalName: www.wolfcode.cn
```

#### 3.2.3 配置外部服务发现(Ingress)

##### 1. 使用 Helm 安装 ingress-nginx

```shell
# 获取二进制安装文件
wget https://get.helm.sh/helm-v3.2.3-linux-amd64.tar.gz

# 解压二进制文件, 将解压目录下的 helm 文件移动至 /usr/local/bin
tar -zxvf xxx
cp helm /usr/local/bin/

# 验证是否安装成功
helm version
```

##### 2. 添加 helm 仓库

```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# 查看仓库列表
helm repo list

# 搜索 ingress-nginx
helm search repo ingress-nginx

# 下载并解压对应安装包
helm pull ingress-nginx/ingress-nginx
tar -xf ingress-nginx-4.12.1.tgz
```

##### 3. 配置参数

```shell
# 修改 values.yaml
registry: registry.cn-hangzhou.aliyuncs.com
image: google_containers/nginx-ingress-controller
image: google_containers/kube-webhook-certgen

hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

kind: DaemonSet
nodeSelector:
  ingress: "true"
  
type: Loadbalacer
# 改为
type： CLusterIP

#
admissionWebhooks:
  enabled: false
```

##### 4. 安装

```shell
# 创建命名空间
kubectl create ns ingress-nginx

# 为节点添加标签
kubectl label ubuntu-new1 ingress=true

# 安装 ingress-nginx
helm install ingress-nginx -n ingress-nginx .
```

##### 5. 使用 ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-example
  annotations:
    kubernetes.io/ingress.classes: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: k8s.wolfcode.cn
    http:
      paths:
      - pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
        path: /api
```

#### 3.2.4 更新回滚

#### 3.2.5 扩缩容

#### 3.2.6 HPA(Horizontal Pod Autoscaler)

##### 1. 部署 deployment 及 service

```yaml
# 部署 deployment, 其中设置 resources 字段
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
  namespace: default
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx-deploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: m.daocloud.io/docker.io/library/nginx
        imagePullPolicy: Always
        name: nginx
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 128Mi
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
# 部署 service, 为已部署的 deployment 提供统一的入口
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx-delpoy
spec:
  selector:
    app: nginx-deploy
  ports:
  - port: 80
    targetPort: 80
    name: web
  type: NodePort
```

##### 2. 启用 HPA

```shell
kubectl autoscale deploy nginx-deploy --cpu-percent=20 --min=2 --max=5
```

##### 3. 管理 HPA

```shell
# 查看 pod 的资源使用情况
kubectl top pod

# 查看 node 的资源使用情况
kubectl top pod
```

##### 4.安装 metrics-server

```shell
# 获取 yaml 文件
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download   /components.yaml -O metrics-server-components.yaml

# 替换其中镜像源
sed -i 's/registry.k8s.io\/metrics-server/registry.cn-hangzhou.aliyuncs.com\/google_containers/g' metrics-server-components.yaml

# 修改 metrics-server-components.yaml 中容器的 tls 配置，使其不验证 tls， 在containers 的 args 参数中增加 --kubelet-insecure-tls 参数

# 安装组件
kubectl apply -f metrics-server-components.yaml
```

##### 5. 测试 HPA

```shell
# 通过死循环向目标 svc 地址发送访问请求
while true; do wget -q -O- http://<ip:port> > /dev/null; done
```






























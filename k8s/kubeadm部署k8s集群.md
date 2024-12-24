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

- /etc/fstab 中注释掉 swap 行以禁用 swap 分区

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

### 5. 安装 kubeadm kubelet kubectl

#### 5.1 For ubuntu

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

- 可选操作

  ```shell
  sudo systemctl enable --now kubelet
  ```

### 6. 配置 cgroup driver

容器运行时以及 kubelet 的 cgroup driver 属性需要匹配，都配置为 systemd

## 二、使用 kubeadm 创建集群

### 1. 初始化控制平面节点

执行指令初始化控制平面节点

```shell
kubeadm init <args>
```

#### ```control-plane-endpoint```

当未设置```control-plane-endpoint```时，创建的单节点控制平面无法在后续的配置中修改成为高可用性的集群。因此若要创建高可用性集群，在初始化时应设置此项，并在后续的配置中修改```cluster-point```指向负载均衡服务器的地址

#### ```--pod-network-cidr```

根据选用的 Pod 网络插件，核实是否需要在 kubeadm init 时传递其他参数。根据第三方插件的不同，可能需要设置```-pod-network-cidr```

#### ```--cri-socket```

kubeadm 会尝试检测容器运行时，但当节点提供多个运行时时，需要指出使用哪一个容器运行时

#### ```--apiserver-advertise-address```

```--apiserver-advertise-address``` 仅能设置当前节点的 API 服务地址，而 ```--control-plane-endpoint```  可设置整个控制平面节点的共享端口

#### ```--upload-certs```

设置此项将上传所有在集群控制平面中共享的证书。如果希望在控制平面节点之间手动复制证书则不要设置此项。

#### ```--config```

导出默认的配置文件以供修改。设置此项与 ```--certificate-key``` 冲突，若使用配置文件的方式初始化集群，需手动在 ```InitConfiguration``` 及 ```JoinConfiguration: controlPlane``` 中添加 ```certificateKey``` 字段。

```shell
kubeadm config print init-defaults > kubeadm-config.yaml
```

测试环境用到的配置文件如下

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

### 2. 安装 Pod 网络插件

需要部署一个基于 CNI 的 Pod 网络插件，这样 Pod 之间才可以互相通信，集群的 DNS(CoreDNS) 服务将在网络插件安装后启动

#### 2.1 Flannel

- 应用配置文件部署 Flannel，使用 Flannel 时应设置 ```--pod-network-cidr```与 Flannel 配置 ```podCIDR``` 相同

  ```shell
  kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
  ```

- 若需要自定义配置，如 ```podCIDR``` (非 ```10.244.0.0/16```)，将文件下载至本地并做相应修改，如修改 flannel 所使用的镜像源

- 插件存在启动错误时可能需要重启 containerd

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

  


## Docker Swarm

### Swarm模式概述
[Swarm mode overview](https://docs.docker.com/engine/swarm/)

Swarm模式是管理Docker daemons集群的高级功能。使用Docker CLI创建swarm集群，为swarm集群部署应用服务，或是对swarm集群进行管理  

一个Swarm服务由多个运行在Swarm模式下的Docker主机组成，这些主机可以是管理者，管理其他成员及进行任务指派，
也可以是工作者，运行Swarm服务。一个Docker主机可以既是管理者又是工作者。  

当创建一个服务时，可以指定他的最优状态：副本个数、网络、存储资源、需要暴露的端口等。Docker将维持这个期望状态。
例如，当某个工作节点故障，Docker会将该节点的任务调度至其他节点上运行，任务作为一个Swarm服务的一部分，是一个运行中的容器，
与独立容器不同的是，这个容器是由Swarm管理者管理的。

当Docker运行在Swarm模式下时，其中工作的各个主机仍然可以创建独立容器，或是其他Swarm服务。

**定义和运行Swarm服务栈(Swarm service stack)的方法与使用Docker Compose定义和运行容器的方法相同。**


#### Swarm的一些特性  

- swarm由内置Docker Engine
- 去中心化设计  
- Declarative service model
- Scaling
- Desired state reconciliation
- Multi-host networking
- Service discovery
- Load balancing
- Secure by default
- Rolling updates

### Swarm术语
[Swarm mode key concepts](https://docs.docker.com/engine/swarm/key-concepts/)

#### 节点(Nodes) 

节点是一个加入到Swarm集群中的Docker引擎实例，也可以看作是一个Docker节点。在单个操作系统上可以运行一个或多个节点，
但在通常情况下Swarm集群部署时，这些节点是工作在许多不同的物理机或云服务器上的。

将服务的定义文件提交给管理节点以将应用部署至Swarm集群中，管理节点将以任务为单位将应用程序分派至各个工作节点。

管理节点也会执行必要的编排以及集群管理功能来确保Swarm集群运行在期望的工作状态下。管理节点之间将选举出一个领导者来执行编排任务。
一般推荐部署奇数个管理节点来充分利用Swarm模式的错误容忍机制。当有多个管理节点时，Swarm集群可以从管理节点的故障中恢复而无须停机。   
Docker推荐的最大管理节点个数为7。

工作节点接收并执行从管理节点分派过来的任务，默认情况下，管理节点也会作为工作节点来运行服务，但用户可以配置管理节点仅作为管理节点，即只运行管理任务。   
每个工作节点上将运行一个代理，报告当前被分配给该节点的任务给管理节点，以实现集群的编排管理。

#### 服务和任务(Services and tasks) 

服务定义了运行在节点上的任务。是Swarm系统的中心结构，也是用户与Swarm交互的主要根源。通常情况下，服务是某些大小应用语境下的微服务镜像。   
当创建一个服务时，需要指定容器的镜像以及需要在容器中运行的命令，此外也可以定义一些其他的设置，例如端口、overlay网络、CPU或内存使用限制、滚动更新策略、副本数等。   
在副本服务模型下，Swarm管理者将基于定义好的期望规模，在节点间分配一定数量的重复任务。对全局服务，Swarm将在集群中的每个可用节点上都运行一个该服务的任务。

任务是一个运行中的容器，任务是Swarm调度的最小原子单位。当一个任务被分配给了一个节点，它将不能被移动至其他节点，它只能在被分配的节点中运行直至退出。


#### 负载均衡(Load balancing) 

The swarm manager uses ingress load balancing to expose the services you want to make available externally to the swarm.

......

### Swarm部署流程

**1. 创建管理节点**  
`docker swarm init --advertise-addr <MANAGER-IP>`    
**2. 查看信息**   
`docker info`   
`docker node ls`   
`docker swarm join-token worker`   
**3. ssh登录到远程主机，将其添加为Swarm节点**   
`docker swarm join`   
**4. 将服务部署至Swarm**   
`docker service create --replicas 1 --name helloworld alpine ping docker.com`   
**5. 查看Swarm上的服务**   
`docker service inspect --pretty <SERVICE-ID>`   
`docker service ps <SERVICE-ID>`   
**6. 扩展Swarm中的服务**   
` docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>`  
**7. 删除运行在Swarm中的服务**   
`docker service rm helloworld`   
**8. 对运行中的服务进行更新**   
`docker service update`   
**9. 配置运行中的节点**   
`docker node update --availability drain <NODE-ID>`  通常被用来配置给管理节点使其不作为工作节点  
`docker node update --availability active <NODE-ID>`   

**Swarm模式路由组网**   
[Use Swarm mode routing mesh](https://docs.docker.com/engine/swarm/ingress/)   
Docker引擎的Swarm模式可以使服务很方便地将他们的端口发布以供Swarm之外的资源访问。所有节点都会加入一个ingress路由网络。
这个路由网络可以使Swarm集群中每个节点都能接收关于其他任意Swarm集群中服务所发布端口的连接访问，即使该节点中没有运行任何任务。
这个路由网络会将所有到来的访问请求转发至可访问节点的活动容器的对应端口。   
默认情况下，Swarm服务使用路由组网发布服务端口。当某个连接访问到一个节点发布的端口时，它会被透明地重定向至一个运行该服务的工作节点。
事实上，Dockers充当了Swarm服务的负载均衡器。   
也可以为Swarm服务配置一个外部的负载均衡器。这种情况下，可以选择使用路由网络或是干脆不使用路由网络。

**Swarm security with public key infrastructure(PKI)**  
[Manage swarm security with public key infrastructure (PKI)](https://docs.docker.com/engine/swarm/how-swarm-mode-works/pki/)   
Docker内置的Swarm模式公钥基础设施(PKI)系统可以轻松安全地部署容器编排系统。Swarm集群中的节点使用mutual Transport Layer Security(TLS)来
验证、授权、对其他节点进行加密通信。   

当通过`docker swarm init`创建Swarm集群时，Docker将其自身设置为一个管理节点。管理节点默认将生成一个根CA证书和一个密钥对，
被用来加密与加入到Swarm集群中其他节点的通信。也可以使用命令`docker swarm init --external-ca`指定一个自己的外部生成的root CA。

管理节点也会生成两个令牌(token)，当需要添加额外的节点至Swarm集群时可以使用，一个worker令牌和一个manager令牌。每一个令牌都包含了
根CA证书的摘要和随机生成的密钥。当一个节点要加入Swarm集群，它使用摘要来验证远程管理者发送过来的根CA证书。远程管理者使用密钥来确保加入的节点是被认可。

### 将栈(Stack)部署至Swarm
[Deploy a stack to a swarm](https://docs.docker.com/engine/swarm/stack-deploy/)   



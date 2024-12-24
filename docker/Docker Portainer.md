## Docker Portainer

Portainer由两部分组成，Portainer Server及Portainer Agent，这两部分都以轻量化容器的形式运行在Docker引擎上。


### Docker Standalone Deployment

#### 部署前提
- 开放服务器的TCP端口9443(or 30779 for Kubernetes with NodePort)以使用UI及API功能
- 开放服务器的TCP端口8000(or 30776 for Kubernetes with NodePort)以使用边缘代理
- 开放代理端的TCP端口9001(or 30778 for Kubernetes with NodePort)以使其能被服务器访问


### Docker Swarm Deployment
[Install Portainer CE with Docker on Linux](https://docs.portainer.io/start/install-ce/server/docker/linux)   
[Install Portainer Agent on Docker Standalone](https://docs.portainer.io/admin/environments/add/docker/agent)   
[Connect to the Docker API](https://docs.portainer.io/admin/environments/add/docker/api)   
[Connect to the Docker Socket](https://docs.portainer.io/admin/environments/add/docker/socket)   
[Install Edge Agent Standard on Docker Standalone](https://docs.portainer.io/admin/environments/add/docker/edge)


>
> > 创建Portainer Server使用数据卷   
> > > `docker volume create portainer_data`
> 
> > 运行Portainer Server容器
> > > `docker run -d -p 8000:8000 -p 9443:9443 
--name portainer --restart=always 
-v /var/run/docker.sock:/var/run/docker.sock 
-v portainer_data:/data portainer/portainer-ce:latest`
> 
> > 进入UI
> > > 访问https://ip-address:9443以进入Portainer Server管理界面
>
> > 部署Portainer Agent
> > > 使用Agent时默认是通过Unix sockets来接入Docker，Agent不支持通过TCP方式连接Docker引擎。
> > 
> > > 在UI内执行相关添加指令并根据提示进行操作   
> >
> > > 如果想要使用Agent的特性[host management feature](https://docs.portainer.io/user/docker/host/setup#enable-host-management-features)
> > > 则需要将必要的数据卷也挂载至Agent容器的内部：   
> > > `-v /:/host`
> 
> > 通过Docker API连接其余Docker主机
> > > 使用此功能需要确保Docker主机被配置成允许远程连接，参考Docker文档进行此配置。
> > 
> > > 在UI内执行相关添加指令并根据提示进行操作   
> 
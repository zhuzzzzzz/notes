## Docker Compose

### Docker Compose如何工作

Docker Compose通过配置compose.yaml(或compose.yml)文件来启动，yaml文件中定义了多容器的Compose应用。  
compose.yaml文件可以通过拓展(extend)、合并(merge)、引用(include)操作来根据指定的文件顺序拓展或覆盖YAML元素。
用户可以通过Compose CLI来与Compose应用进行命令交互。

### compose.yaml主要组成元素

- version  
用以定义Compose文件编译生成时的版本，仅作提示性功能
- name  
用以定义Compose文件的项目名称。  
项目，是平台上对一个特定程序的一个单独部署单元。通过设置项目名称，可以用来对资源进行分组
- services  
应用程序组件定义为services，服务(service)是一个以相同配置在平台上多次运行同一个容器镜像的抽象概念
- networks  
- volumes
- configs
- secret

### Docker Compose操作

[docker compose文档](https://docs.docker.com/storage/volumes/)

- `docker compose -f COMMAND` 使用一个或多个 `-f` 命令以指定Compose文件
(当指定多个Compose文件时，所有的文件路径都是关于首个指定文件的相对路径)


- `docker compose -p COMMAND` 指定Compose项目名称，名称设置的优先级如下:
   1. 通过-p命令指定的名称
   2. COMPOSE_PROJECT_NAME环境变量 
   3. Compose文件中指定的名称 
   4. Compose文件的目录名称 
   5. 当前工作路径的目录名称


- `docker compose --profile COMMAND` 指定一个或多个profiles


- `docker compose up -d` 对服务进行编译、创建、并分配容器。此命令也会启动所有被连接的、未启动的服务。  
  此命令会聚合所有的容器的输出(类似于`docker compose logs --follow`)，
  可以通过设置`--attach`或`--no-attach`(或设置Compose文件中service的attach字段)来选择收集或不收集这些日志。  
  若对某服务，已经存在运行的容器了，但是服务的配置或是镜像在容器创建后发生了改变，此命令会挑出改变的部分，
  停止并重新创建对应的容器(保存已经挂载的volumes)，要阻止这种自动更新，使用`--no-recreate`，若要强制使Compose项目更新，
  使用`--force-recreate`。


- `docker compose down -v`   


- `docker compose exec` 对Compose service进行操作，等价于docker exec  
通过这个子命令，用户可以在某个服务中运行任意命令，比如通过`docker compose exec web bash`来获得一个与该服务的交互终端。


- `docker compose ls` 列出当前运行的所有Compose项目  


- `docker compose ps` 针对某个Compose项目列出所有容器  


- `docker compose top` 针对某个Compose项目列出所有正在运行的进程  


- `docker compose inspect` 针对某个Compose项目列出所有相关资源














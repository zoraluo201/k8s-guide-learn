# Kubernetes 实战指导（更新中）

本文以一个简单Go应用为例，演示如何一步步在生产环境中使用Kubernetes。

## 1. 部署一个完整的应用

### 1.1 编写一个简单的Go应用

- [main_multiroute.go](k8s_actions_guide/version1/go_code/main_multiroute.go)

这个Go应用的逻辑很简单，它是一个支持从配置动态加载路由的HTTP服务器。初始化此应用：

```shell
go mod init k8s_action
go mod tidy
```

### 1.2 使用ConfigMap存储配置

传统方式下，我们通常将配置文件存储在单独文件中，并通过环境变量或者命令行参数传递给应用。

在K8s环境中，我们需要将配置迁移到ConfigMap中，并通过环境变量或者卷挂载的方式传递给应用。

- [base_manifest/configmap.yaml](k8s_actions_guide/version1/base_manifest/configmap.yaml)

注意将K8s清单放在一个单独的目录（如`base_manifest`）下，以便后续批量部署。

> 虽然可以在Dockerfile中直接将配置文件打包到容器，但这种方式通常伴随的是将配置文件存储在代码库中，这并不符合K8s的最佳实践。
> 同时也不适合用来存储重要配置。

如果有重要的配置，比如证书私钥或Token之类的敏感信息，请使用Secret来存储。

### 1.3 使用Secret存储敏感信息

通常一个后端应用会链接到数据库来对外提供API服务，所以我们需要为应用提供数据库密码。

虽然ConfigMap也可以存储数据，但Secret更适合存储敏感信息。在K8s中，Secret用来存储敏感信息，比如密码、Token等。

- [base_manifest/secret.yaml](k8s_actions_guide/version1/base_manifest/secret.yaml)

#### 1.3.1 加密存储Secret中的数据

虽然Secret声称用来存储敏感信息，但默认情况下它是非加密地存储在集群存储（etcd）上的。
任何拥有 API 访问权限的人都可以检索或修改 Secret。

请参考以下链接来加密存储Secret中的数据：

- [K8s Secrets](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/)
- [加密 K8s Secrets 的几种方案](https://www.cnblogs.com/east4ming/p/17712715.html)

### 1.4 使用Dockerfile打包应用

这一步中，我们编写一个Dockerfile文件将Go应用打包到一个镜像中，以便后续部署为容器。

- [Dockerfile](k8s_actions_guide/version1/go_code/Dockerfile)

注意在Dockerfile中定制你的Go版本。

### 1.5 准备镜像仓库

为了方便后续部署，我们需要将打包好的镜像上传到镜像仓库中。

作为演示，本文使用Docker Hub作为镜像仓库。但在生产环境中，为了应用安全以及提高镜像拉取速度，我们应该使用（搭建）私有的仓库。

常用的开源镜像仓库有Docker Registry和Harbor。如果你使用云厂商的托管集群，可以使用它们提供的镜像仓库产品。

### 1.6 编写Deployment模板

Deployment是K8s中最常用的用来部署和管理应用的资源对象。它支持应用的多副本部署以及故障自愈能力。

- [base_manifest/deployment.yaml](k8s_actions_guide/version1/base_manifest/deployment.yaml)

你可以在模板中定制应用的副本数量、资源限制、环境变量等配置。

> 注意：你可能看情况需要修改模板中的namespace，生产环境不建议使用default命名空间。因为这不利于对不同类型的应用进行隔离和资源限制。
> 比如，你可以为后端服务和前端服务分别使用backend和frontend命名空间。

**为镜像指定指纹**  
docker拉取镜像时支持使用如下命令：

```shell
docker pull busybox:1.36.1@sha256:7108255e7587de598006abe3718f950f2dca232f549e9597705d26b89b7e7199
# docker images --digests 获取镜像hash
```

后面的`sha256:710...`是镜像的唯一hash。当有人再次推送相同tag的镜像覆盖了旧镜像时，拉取校验就会失败，这样可以避免版本管理混乱导致的部署事故。

所以我们可以在Deployment模板中指定镜像的tag的同时使用`@sha256:...`来指定镜像的hash以提高部署安全性。

### 1.7 使用CI/CD流水线

完成前述步骤后，应该得到以下文件布局：

```
├── Dockerfile
├── go.mod
├── go.sum
├── base_manifest
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── secret.yaml
└── main_multiroute.go
```

现在可以将它们提交到代码仓库中。然后使用CI/CD流水线来构建镜像并部署应用。

> 代码库中通常会存储***非生产环境**的配置文件，对于生产环境使用的配置文件（ConfigMap和Secret），不应放在代码库中，
> 而是以手动方式提前部署到环境中。

参考下面的指令来配置CI/CD流水线：

```shell
# 假设现在已经进入到构建机（需要连接到k8s集群）

IMAGE=leigg/go_multiroute
TAG=v1 # 每次迭代时手动指定
_IMAGE_=$IMAGE:$TAG

# 构建镜像（将leigg替换为你的镜像仓库地址）
$ docker build . -t $_IMAGE_
...
Successfully built 20e2a541e835
Successfully tagged leigg/go_multiroute:v1

# 推送镜像到仓库
$ docker push $_IMAGE_
The push refers to repository [docker.io/leigg/go_multiroute]
f658e2d998f1: Pushed
06d92acd05c8: Pushed
3ce819cc4970: Mounted from library/alpine
v1: digest: sha256:74bf6d94ea9af3e700dfd9fe64e1cc6a04cd75fb792d994c63bbc6d69de9b7ee size: 950

# 部署应用
$ kubectl apply -f ./base_manifest

# 更新应用
$ kubectl set image deployment/go-multiroute go-multiroute=$_IMAGE_
```

查看应用部署情况：

```shell
$ kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
go-multiroute   2/2     2            2           7s
$ kubectl get po    
NAME                            READY   STATUS    RESTARTS   AGE
go-multiroute-f4f8b64f4-564qq   1/1     Running   0          8s
go-multiroute-f4f8b64f4-v64l6   1/1     Running   0          8s
```

这里使用的是简单的滚动更新策略进行部署更新应用，实际环境中你可能会根据应用情况而使用蓝绿部署或金丝雀部署，
这部分将在本文的[5.8 部署策略](doc_k8s_actions_guide.md#58-部署策略)中进行详细讨论。

### 1.8 为服务配置外部访问

现在已经在集群内部部署好了应用，但是还无法从集群外部访问。我们需要再部署以下资源来提供外部访问能力。

- Service：为服务访问提供流量的负载均衡能力（支持TCP/UDP/SCTP协议）
- Ingress：管理集群外部访问应用的路由端点，支持HTTP/HTTPS协议
    - Ingress需要安装某一种Ingress控制器才能正常工作，常用的有Nginx、Traefik。

> 如果应用是非HTTP服务器（如仅TCP、Websocket服务），则无需Ingress，仅用Service来暴露服务就可。

- [expose_manifest/service.yaml](k8s_actions_guide/version1/expose_manifest/service.yaml)
- [expose_manifest/ingress.yaml](k8s_actions_guide/version1/expose_manifest/ingress.yaml)

部署Ingress控制器的步骤这里不再赘述，请参考[基础教程](doc_tutorial.md#82-安装Nginx-Ingress控制器)。

下面是部署Service和Ingress的步骤：

```shell
$ kubectl apply -f ./expose_manifest
ingress.networking.k8s.io/go-multiroute created
service/go-multiroute created

# 注意：service/kubernetes是默认创建的，不用理会
$ kubectl get svc,ingress                           
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/go-multiroute   ClusterIP   10.96.219.33   <none>        3000/TCP   19s
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP    6d

NAME                                      CLASS   HOSTS   ADDRESS   PORTS   AGE
ingress.networking.k8s.io/go-multiroute   nginx   *                 80      19s
```

现在，可以通过Ingress控制器开放的端口来访问应用了。笔者环境安装的Nginx Ingress控制器，查看其开放的端口：

```shell
# PORT(S) 部分是Nginx Ingress控制器内对外的端口映射
# 内部80端口映射到外部 30073，内部443端口映射到外部30220
kubectl get svc -ningress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.171.227   <pending>     80:30073/TCP,443:30220/TCP   3m46s
ingress-nginx-controller-admission   ClusterIP      10.96.7.58      <none>        443/TCP                      3m46s
```

使用控制器的端口访问服务：

```
# 在任意节点上进行访问
$ curl 127.0.0.1:30073/route1
[v1] Hello, You are at /route1, Got: route1's content
 
# /route2 尚未在ingress的规则中定义，所以不能通过ingress访问
$ curl 127.0.0.1:30073/route2
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

OK，现在访问没有问题了。

#### 1.8.1 为什么通过30073端口访问

Nginx Ingress控制器默认通过NodePort方式部署，所以会在宿主机上开放两个端口（本例中是30073和30220），
这两个端口会分别代理到Ingress控制器内部的80和443端口。

本例中部署的Go应用是一个后端服务，对于向外暴露的端口号没有要求。
但如果是一个前端应用，比如是一个Web网站，那么可能就有对外暴露80/443端口的要求。
此时就需要调整Ingress控制器部署方式，使用LoadBalancer或`HostNetwork`方式部署。

### 1.9 更新配置的最佳实践

应用上线后，我们可能会有更改应用配置的需求。一般的做法是直接更新现有的ConfigMap，然后重启所有Pod。
但这不是K8s的最佳实践。原因有以下几个：

- 更新ConfigMap不会触发Deployment中所有Pod重启
- 若更新后的配置有问题不能迅速回滚（需要再次编辑现有ConfigMap），导致服务短暂宕机

而最佳实践是为ConfigMap命名带上版本号如`app-config-v1`，然后部署新版本的ConfigMap，
再修改Deployment模板中引用的ConfigMap名称，最后更新Deployment（触发所有Pod滚动更新）。

当需要回滚时，再次更新Deployment即可。

## 2. 开发者工作流

为了提高开发人员的工作效率和最大化在日常工作中模拟在生产环境中开发，我们通常会搭建一个开发集群来提供给开发者们完成日常开发。
但随之而来的就是集群的用户管理、资源额度限制以及回收工作，我们可以开发脚本来自动化完成这些工作。

### 2.1 搭建不同环境的集群

**开发集群**  
我们需要一个共享K8s集群来提供给开发者们使用，他们的日常开发流程包括PR提交、代码review、构建、部署等工作都将和生产环境保持基本一致。

但我们必须为每个开发者分配单独的K8s命名空间，然后设置资源额度限制，防止开发者们无节制地使用集群资源以造成互相干扰。
当团队规模较大时，建议为10~20人共享一个集群，而不是上百人共享一个超大集群，这样可以简化集群的管理工作。

**测试环境**  
当开发者在开发环境完整测试过自己提交的代码后，就可以考虑部署到测试环境，并告知测试人员进行测试。
测试环境通常只有一个K8s集群由所有人共享。

**预发布环境**  
大部分公司都会有一个预发布环境，用来部署一些需要经过测试环境验证的应用版本，比如一些新功能或一些需要修复的bug。
这个环境应该与生产环境尽可能保持高度一致，包括节点数量、网络等其他配置，且相对测试环境具有更严格的权限控制。
不允许开发者随意更改数据库等配置，测试人员在预发布环境应按照生产环境的标准进行测试。

**镜像仓库**  
镜像仓库是用来存放应用镜像的地方，通常由运维人员来维护。每个环境都应该有一个本地的镜像仓库，以加快应用部署。

### 2.2 管理员使用的脚本

为了简化集群的用户管理、资源额度限制以及回收工作，我们可以开发一些脚本来协助管理员轻松完成这些工作。

- [new_user.sh](k8s_actions_guide/version1/script/new_user.sh)：在集群中添加一个新用户，并创建相应的命名空间、角色和额度限制。
    - 此脚本会同时生成`client-cert-$USER.crt`和`client-cert-$USER.key`
- [del_user.sh](k8s_actions_guide/version1/script/del_user.sh)：删除集群中的一个用户命名空间（包含空间下的所有资源）。
- [setup_kubeconfig.sh](k8s_actions_guide/version1/script/setup_kubeconfig.sh)：初始化用户使用的kubeconfig文件。
    - 此脚本会在执行目录下生成`dev-config`文件，将此文件分发给对应开发人员作为kubeconfig文件即可。

> 笔者在脚本中添加了详实的注释，你可以阅读它们来了解脚本用法和步骤含义。

### 2.3 更方便的查询应用日志

应用上线后，我们通常会有查看应用日志的需求。
默认的应用日志分散存储在每个Pod所在节点的`/var/log/pods`目录，且它们会伴随Pod的消失而被删除。
这不利用开发者们查看日志的需求，因此我们需要一个中心化存储日志的方案。
请参考[Kubernetes 日志收集](doc_log_collection.md)来了解如何搭建日志收集系统。

如果因为某些原因不希望搭建日志收集系统（或者说是偷懒~），
你也可以通过修改每个节点上的kubelet配置来调整容器日志的存储大小和文件数量，具体步骤也请参考[Kubernetes 日志收集中的
*业务容器日志*](doc_log_collection.md#11-业务容器日志)。
其次，你还可以参考[Kubernetes 维护指导中的*日志查看*](doc_maintaintion.md#15-日志查看)来使用第三方工具来高效查询容器日志。

> 除了Pod日志，我们还需要关注集群组件日志、节点日志、集群审计日志。

### 2.4 启动开发

开发过程中，开发者需要频繁地构建并推送镜像、更新应用、查看应用日志。
这其中最为重要的步骤就是镜像tag定义以及更新应用的操作。

**镜像tag使用版本化语义而不是latest**  
在构建用于开发环境的自测试镜像时，对于镜像tag，开发者最好是使用版本化语义而不是`latest`。
使用`latest`可以免去每次构建镜像时都需要手动修改tag的麻烦，但同时也带来了不确定性。
因为你无法完全确定所更新的应用使用了你刚刚构建并推送的镜像。倘若你将误以为自测成功的代码发布到了生产环境，
难以想象将会发生的事情和面临的后果😑。

**更新应用**  
在使用版本化语义的镜像tag后，我们可以更从容的使用`kubectl set image...`命令来更新应用。
注意，在开发以及后续的上线过程中，我们都不需要修改代码库中的Deployment YAML文件。

### 2.5 使用第三方工具为开发提效

#### 2.5.1 IDE插件

编写此文时已经是2024年了，主流的VSCode和Jetbrains家族IDE都已经有大量的Kubernetes插件可用，
这些插件可以帮助开发者通过图形化的方式与开发环境的K8s集群进行便捷交互，免去手动输入kubectl命令的繁琐。

#### 2.5.2 K9s终端面板

K9s是一个终端形式的K8s资源面板，它支持在终端中以可视化方式查看和管理Kubernetes集群中的资源，包括Pod、Deployment、Service等。
它支持以方向键、回车键和空格键与面板交互，免去手敲命令的麻烦。

我们可以在测试环境和预发布环境（也包括生产环境）安装K9s，这样可以更便捷的查看应用状态、日志、事件、以及进入Pod内的容器Shell，极大地改善了K8s的使用体验。

#### 2.5.3 开源K8s日志工具

常规查看容器日志的命令是`kubectl logs`，但这个命令有一些局限性，比如：

- 一次只能查看一个Pod的日志（若不使用`-l`的话）
- 不能指定Deployment、Service、Job、Stateful和Ingress名称进行日志查看
- 不能查看指定节点上的所有Pod日志
- 不支持颜色打印

等等。你可以使用开源的K8s Pod日志查看工具来提高效率，具体可以参考 *Kubernetes 维护指导*
中的[日志查看](doc_maintaintion.md#15-日志查看)小节。

#### 2.5.4 K8s资源清单风险分析工具

或许在较小的集群中或者是不那么重要的业务中，我们并不会去特别注意K8s资源清单的编写规范，例如对容器的CPU/内存限制、设置容器安全上下文等。
但我们需要知道，Pod中的容器是本质上来说还是运行在集群节点上的，不安全的资源清单可能会导致容器在节点上被恶意利用，从而导致集群被攻击。

好在已经有不少开源工具可以协助我们轻松完成资源清单的修复工作，这里推荐以下几个工具：

- [KubeLinter](https://github.com/stackrox/kube-linter)
- [KubeSec](https://kubesec.io/)
- [kube-score](https://github.com/zegl/kube-score)
- [polaris](https://github.com/FairwindsOps/polaris)

你可以将这些工具中的一个或几个添加CI/CD流水线中，以在每次提交代码时自动检查资源清单的安全性。

## 3. 采集集群指标

### 3.1 简介

指标是指针对某一种资源在一段时间内的使用情况的统计，例如CPU使用率、内存使用量、网络带宽、磁盘使用量等。

指标采集通常有两种说法，即黑盒监控和白盒监控。黑盒监控是从集群的外部视角采集数据。多用于传统的CPU、内存、磁盘等硬件资源的监控，
非常适用于对基础设施的监控。而白盒监控更关注应用程序的状态细节，比如HTTP请求总数、500错误请求数和请求延迟等。
白盒监控让我们知道系统为什么处于当前状态，让故障定位进一步有迹可循。

### 3.2 两种监控模式

它们分别是USE和RED模式。

#### 3.2.1 USE模式

USE的释义如下：

- U——Utilization（利用率）
- S——Saturation（饱和度）
- E——Errors（错误率）

这种模式专注于基础设施监控，对应黑盒监控。

#### 3.2.2 RED模式

RED的释义如下：

- R——Rate（每秒接受的请求数）
- E——Error（每秒失败的请求数）
- D——Duration（每个请求的耗时）

这种模式专注于应用程序监控，对应白盒监控。

### 3.3 采集目标

了解了上面的监控模式后，现在我们需要知道应该在集群中进行指标采集的目标有哪些。
这里笔者将它们分类列出：

- 控制平面：API Server、etcd、Scheduler、Controller Manager
- 工作节点：kubelet、容器运行时、kube-proxy、kube-dns和Pod

上面除了工作节点的Pod以外，其他可以归类为基础设施组件。我们需要监控这些组件暴露的各项指标并及时做出响应，
才能确保集群的稳定运行。

### 3.4 采集架构

#### 3.4.1 使用Prometheus作为存储后端

Prometheus是CNCF（云原生计算基金会）中排名仅次于 Kubernetes 的一个重量级开源项目，是一个用于监控和告警的开源系统。
它提供了一个灵活的查询语言叫做**PromQL**，让我们可以方便地查询和分析监控数据。目前，Prometheus已经是业界公认的监控事实标准。

Prometheus架构图如下
![](./img/prometheus_architecture.png)

简单来说，Prometheus由以下几个部分组成：

- Prometheus Server：负责数据采集和存储，并提供PromQL查询语言的支持。
- Push Gateway：支持临时性任务的数据采集。
- Exporter：用于暴露被监控组件的数据接口。
    - 对于不同的采集目标（例如主机节点）需要部署对应的Exporter，然后配置Prometheus主动采集即可。
    - 常见的有：node_exporter、blackbox_exporter、mysqld_exporter、redis_exporter等。
- Client Library：客户端库，为需要监控的组件提供方便的接入方式。
    - 对于那些没有Exporter的采集目标（比如业务应用），我们可以通过客户端库自行上报数据到Prometheus Server中。
- Alert-manager：负责接收Prometheus的告警信息，并决定如何对告警进行处理，如发送邮件、短信、调用Webhook等。

通过上面的架构图和文字说明，我们可以了解到Prometheus支持以推/拉的方式采集各种目标提供的指标数据。

#### 3.4.2 Prometheus四种指标类型

Prometheus中的指标可以分为以下四种类型：

- Counter（计数器）
    - 简介：Counter 是一个累加器，只能增加，不能减少。通常用于表示累积的事件计数，比如请求总数、错误总数等。
    - 典型用法：记录事件的总数量，例如 HTTP 请求总数、错误数量、任务完成次数等。
    - 示例：http_requests_total, errors_total, tasks_completed_total。
- Gauge（仪表盘）
    - 简介：Gauge 是一个可变化的数值，可以增加也可以减少。用于表示可变的度量，如温度、内存使用率等。
    - 典型用法：跟踪随时间变化的指标，例如 CPU 使用率、内存占用量、连接数等。
    - 示例：cpu_usage, memory_usage, active_connections.
- Histogram（直方图）
    - 简介：Histogram 统计和存储数据的分布情况，如请求响应时间的分布。
    - 典型用法：衡量持续时间或值的分布情况，例如请求响应时间、API 调用耗时等。
    - 示例：http_request_duration_seconds.
- Summary（摘要）
    - 简介：Summary 也用于记录持续时间数据，但它提供的是可变精度的摘要，而不是固定数量的桶。
    - 典型用法：与 Histogram 类似，用于记录持续时间，但通常用于更复杂的分布情况，比如 p50、p90、p99 等分位数。
    - 示例：api_request_duration_seconds_summary.

这四种指标类型几乎覆盖所有场景，并且每种类型都提供了丰富的标签（label）用于描述指标的维度信息。
我们只需要在推/拉数据时指定需要采集的指标类型和标签，Prometheus就能自动进行数据采集和存储。

#### 3.4.3 使用Grafana作为可视化组件

Prometheus本质上只是一个时序数据库，它本身并不具备强大的可视化能力。要想将采集到的指标数据进行丰富的可视化展示，
我们需要使用一个可视化组件，它就是Grafana。Prometheus+Grafana是一个常见的兄弟组合，几乎不会分开使用。

Grafana是一个开源的度量分析和可视化平台，它可以通过将时序数据导入其中而建立一个数据仪表盘。想象一下，
你只需要通过一个网页上的数据大盘就能对整个集群（包括几十上百甚至更多的节点）的运行状态了如指掌，这该是多么酷的一件事情。

当然这个兄弟组合并不仅仅用于Kubernetes集群监控，它还可以用于各种需要监控和可视化的场景。比如在你的业务场景中，
需要监控今/昨日的营收、昨日的PV、今日的UV、今日的订单量等。

#### 3.4.4 采集容器指标（cAdvisor）

cAdvisor是Google开源的一款用于展示和分析容器运行状态的可视化工具。通过在主机上运行CAdvisor用户可以轻松的获取到当前主机上容器的运行统计信息，
例如CPU使用率、内存使用量、网络带宽和磁盘使用量等。你可以参考 [cadvisor的安装与使用][cadvisor] 来进一步了解它的基本原理和使用方法。

cAdvisor暴露了Prometheus支持的指标格式，通过二者结合，我们可以轻松获取到Kubernetes集群中的Pod内部容器级别的监控数据。

> kubelet也是通过内置cAdvisor来监控容器指标。

#### 3.4.5 Metrics Server

Metrics Server是K8s的一个附加组件，它实现了API Server定义的[Metrics API][MetricsAPI]。
Metrics API主要为用户提供集群中处于运行状态的Pod和Node的CPU和内存使用情况，
设计用于K8s的HPA（Horizontal Pod Autoscaling，Pod水平自动伸缩）以及VPA（Vertical Pod Autoscaling，Pod垂直自动伸缩）功能。

Metrics Server内部通过调用kubelet API来监控容器数据，然后通过Metrics API暴露给API Server使用。
当安装Metrics Server后，我们可以使用`kubelet top`命令来查看集群中Pod和Node的CPU和内存使用情况。关于它的安装和使用细节，
你可以参考笔者的另一篇文章*K8s进阶教程*中的 [安装Metrics Server插件](doc_tutorial_senior.md#341-安装Metrics-Server插件)
一节。

了解更多：

- [Resource Metrics Pipeline](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Metrics Server](https://github.com/kubernetes-incubator/metrics-server)

#### 3.4.6 自定义指标

前面我们说到使用Prometheus+Grafana的组合来监控Kubernetes集群，这种方式已经可以监控任何的指标数据。
如果我们想要把Prometheus中存储的指标数据通过暴露给Kubernetes API Server，然后通过kubectl命令行来查询，
那我们可以通过自定义指标的方式来完成，这需要在集群中安装**prometheus-adapter**，
并在其配置文件中编写需要查询的指标信息，大致步骤请参考*K8s进阶教程*
中的[3.4.6 使用多项指标、自定义指标和外部指标](doc_tutorial_senior.md#346-使用多项指标自定义指标和外部指标)。

但请注意，如果仅仅是为了方便使用kubectl来查询指标，那其实大可不必，因为性价比太低（有一定维护成本），使用Grafana查询足以。
使用自定义指标更多是为了完成HPA（Horizontal Pod Autoscaling，Pod水平自动伸缩）和VPA（Vertical Pod
Autoscaling，Pod垂直自动伸缩）工作。

### 3.5 告警

有了指标数据后，我们需要根据SLO（服务水平目标）来设置相应的告警规则（在Grafana中设置）。SLO是对服务的某些可测量特性设置的目标期望，
例如可用性、吞吐量、频率和响应时间。如果没有SLO，我们对服务就会抱有不切实际的期望，也无法设置合适的告警规则。
对于Kubernetes这样服务具有高度自愈能力的系统，我们应该针对终端用户的服务体验来设置告警规则。
例如，为前端服务设置的SLO是响应时间不得高于20ms，当观测到一段时间内的平均响应时间高于20ms，就需要及时发出告警。

下面是一些常见的注意点以供参考：

- 仅对关键指标进行告警，并忽略一些可以被K8s自动处理的指标，例如CPU/内存利用率等
- 设置合理的阈值周期，避免过短的周期导致频繁告警。最后有一套阈值设置规范，来避免个性化的阈值设置。例如，可以遵循5min、10min、30min、1h这样的特定频率来统一配置阈值周期
- 在配置告警规则时，应该确保通知中包含必要的上下文信息，例如服务名称、告警持续时间、建议处理措施等
- 告警通知不要发给一群人，而是仅发给需要关注或处理问题的人，否则容易被当成不重要的信息而被忽略

## 4. 日志监控

日志监控是监控系统中的重要一环，它可以帮助我们快速定位问题和恢复服务。Kubernetes中的日志监控目标包含：

- 节点日志（节点关键服务的日志。例如容器运行时组件的日志、内核日志等）
- Kubernetes组件日志（如API Server、ControllerManager和Scheduler）
- 容器日志（主要是应用日志）
- Kubernetes审计日志（与权限相关，非常重要）

如果你使用云托管的Kubernetes集群，那建议你也使用托管的日志监控服务，这样有助于大幅降低运维成本。维护自建的日志服务起初看起来不错，
但随着环境复杂度的增长，维护工作会变得越来越费时费力。

如果选择自建日志服务，向你推荐笔者的另一篇文章[_Kubernetes 日志收集_](doc_log_collection.md)。
这篇文章会手把手指导你如何完成集群的日志收集工作。

## 5. CI/CD流水线

CI/CD的目标是构建一个完全自动化的过程，覆盖代码从提交到部署再到生产的全流程。我们应该避免进行手动更新，因为这样做很容易出错，
容易导致配置漂移，还会让部署变得脆弱，导致应用交付的整体敏捷性丧失。

构建一个高度集成的流水线能够让我们信心十足地将应用部署到生产环境，本节将具体介绍这个构建过程。

### 5.1 完整的流水线示例

如下：

- 推送代码变更到代码仓库（Git或SVN）
- 运行APP构建
- 运行APP测试
- 基于测试成功的APP代码构建镜像
- 推送镜像到镜像仓库
- 部署APP到Kubernetes（可能基于不同的策略，如滚动更新、蓝绿部署和金丝雀部署）

### 5.2 代码版本控制

每个CI/CD流水线都始于版本控制，它用于维护应用代码和配置文件的变更记录。配置文件可能包括K8s清单以及Helm Chart。
而版本控制策略则有多种选择，这取决于组织结构和责任划分。但请注意，应该尽量让开发者和运维工程师在同一个代码存储库中进行写作，
这有助于统一管理和保证代码和运维脚本物料始终保持匹配。

### 5.3 持续集成

持续集成（CI）步骤是将代码变更持续地集成到版本控制库的过程。具体来说，
就是将每一次代码提交都当做一个新的应用版本进行构建，这样一来就能及时发现哪一次代码提交导致构建失败，从而快速修复代码。

### 5.4 测试

当应用构建成功后，应该马上执行预设的测试命令。例如，Go应用可以使用`go test`执行一组单元测试。
当测试通过时，才能将代码打包为镜像；一旦测试失败，应立即停止流水线工作，并提供此步骤失败的日志输出。

> 除了对应用的单元测试，你可能还需要预设其他测试脚本命令。例如，对K8s清单文件的风险检测命令以及对Helm Chart文件的lint测试命令。

### 5.5 镜像构建

在完成镜像构建时，我们需要提前写好Dockerfile。这里我们需要注意以下几点：

- 使用多阶段构建减小镜像Size
- 使用具有少量命令的基础镜像以减少安全风险，例如Scratch、Alpine和Distroless等
    - 使用这类基础镜像时，你可能还需要了解它们的调试方法，以更好的在生产环境中使用

### 5.6 添加镜像标签

在将镜像推送到镜像仓库之前，需要给镜像打上标签。合理的镜像标签能够帮助我们快速识别镜像版本，在CI流水线中构建的每个镜像都应该拥有一个唯一的标签。

有以下几种标签策略可供使用：

- 构建号（在开始构建时由构件系统产生的一个递增编号）
- GitHash（代码提交时产生的GitHash码，有助于在代码提交记录中溯源）
- GitHash-构建号（推荐）

### 5.7 持续部署

持续部署（CD）步骤是将应用镜像部署到应用运行环境（如测试、预发布和生产环境）的过程。
这一步我们只需要在CD系统中提前配置好Kubernetes的ServerUrl、证书以及Token，然后由CD系统自动完成部署/更新步骤。

### 5.8 部署策略

本节讨论的部署策略，包括滚动更新、蓝绿部署和金丝雀部署。

#### 5.8.1 滚动更新

滚动更新（rolling update）是Kubernetes中最常用也是Deployment默认的部署更新策略。它通过逐步更新Pod的方式，确保应用在更新过程中始终可用。

具体来说，在准备更新时，我们首先执行`kubectl set image...`命令来修改Deployment中应用容器的镜像标签。
然后，Kubernetes会逐步更新Deployment中的Pod，确保应用在更新过程中始终可用。在更新过程中，
如果由于镜像问题导致新的Pod无法正常启动，Kubernetes会暂停更新，待我们解决问题后重新构建新的镜像，再重新操作此步骤完成更新。
此外，我们还可以下面的kubectl命令来辅助完成滚动更新：

- 通过`kubectl rollout pause`命令来暂停滚动更新，并在确认新Pod正常启动且接受流量后再继续更新。
- 通过`kubectl rollout undo`命令来回滚Deployment的本次更新，执行命令后Deployment内的应用容器的镜像标签将恢复到旧值，
  同时当前更新失败的Pod也会被旧标签的镜像替换。

为了更好的实现热更新以及减少终端用户感知，建议在Deployment中配置`readiness probe`和`preStop`字段。
`readiness probe`用于确保部署的新版本Pod在准备就绪后才能接受流量；而`preStop`用于在Pod被删除前执行清理操作，例如应用进程的退出操作。

滚动更新时，集群中会同时存在新旧两个版本的Pod，所以对于不支持多进程部署的应用（比如Deployment的`replicas`字段设置为1），
注意要在PodSpec中设置更新策略为`Recreate`，而不是使用默认的`rollingUpdate`。

#### 5.8.2 蓝绿部署

蓝绿部署是指在已经存在一套生产环境的应用的情况下，通过部署一套新的应用环境，然后将用户流量快速切换（通过负载均衡器）到新版本的应用上。
这个过程中，旧的环境叫做蓝（blue），新部署的环境叫做绿（green）。当新环境存在问题时，再快速切换到旧环境即可。
蓝绿部署最大的优点是实现零停机部署。

这套策略看似简单，实则存在不少难点。蓝绿部署通常意味着一次切换一整个环境。比如，
当前应用由一个副本数量为10的Deployment环境（如名为`app-deploy-v1`）组成，
那么还需要部署一套新的Deployment环境（如名为`app-deploy-v2`，但`label`不同），副本数量仍然为10。这两套环境是同时存在的，
所以会占用增加一倍的硬件资源。然后我们需要对新的Deployment进行完整的测试，测试通过后，
在Kubernetes的Service模板中修改Selector来切换流量到新的Deployment。待新环境正常运行一段时间后（由具体情况决定），再销毁旧的环境。

这里，简要列出蓝绿部署的几个难点：

- 多版本同时存在的问题。应用必须支持多版本同时存在，否则不适用本部署方法。
- 数据库迁移问题。应用通常会访问数据库，而且需要执行事务。操作者必须考虑到新旧版本对同时执行事务操作的兼容性，确保数据一致性和完整性。
- 需要具备同时维护两套环境的能力。
- 由于更新粒度大，虽然可以快速切换流量，但可能会因为操作失误导致严重的服务中断影响。一定要进行预先的、多次的完整流程操作演练。

#### 5.8.3 金丝雀部署

金丝雀部署是比蓝绿部署更保守一点的部署策略。首先，金丝雀部署策略中，也需要同时部署两个基本相同的环境。最大的不同是，
金丝雀部署中，用户流量是按比例或条件逐渐切换到新环境，而不是像蓝绿部署那样一次性地全部切换。
例如，你可以将携带特定Header键值的请求路由到新环境。

> 在大部分移动应用项目中，客户端人员通常会通过特定的Header键值来标识当前用户使用的包环境，比如测试包/正式包/灰度包。

金丝雀部署相对蓝绿部署的优点是，可以小范围验证新版本的应用，确保新版本的应用在生产环境中没有问题后再完全切换。
即使新环境存在问题，其影响也限制在较小范围。这种发布方式通常也叫做A/B发布或灰度发布。

在Kubernetes中可以通过Ingress来实现按比例分配流量的功能，常用的Ingress控制器如Nginx、Traefik等都支持。
分配流量的策略包括HTTP Header的Key/Value、Cookie、比例配置等。

注意，实践金丝雀部署时需要处理与蓝绿部署相同的问题，包括多版本共存、数据库迁移等。

- [使用 Nginx Ingress 实现金丝雀发布](https://cloud.tencent.com/document/product/457/48907)

#### 5.8.4 使用Helm

Helm是一个流行的Kubernetes应用的包管理工具，我们可以在前面提到的部署策略中用到它。

Helm使用一种叫做Chart的包结构来组织编排Kubernetes应用，
一个Kubernetes应用可包含多个Kubernetes资源（包括Deployment、ConfigMap/Secret、Job等在内的各类资源）。
使用Helm，我们的Kubernetes应用就直接拥有了版本管理机制，并且可以通过Helm命令进行快速的升级和回滚。
此外，Helm最关键的特性是支持通过Go Template支持模板化配置，这个特性让我们无需频繁修改Chart，
而是通过CD命令来将参数注入Chart。

比如，我们由一个Web后端应用，在使用Helm之前，这个应用由两个Deployment组成（比如User和DB两个服务）。
然后假设这两个服务都需要在本次迭代中进行升级，若不使用Helm，则需要一个一个升级（执行两次升级操作）；
若使用Helm，这两个Deployment就同时存在于一个Chart中，我们直接编辑好新版本的Chart，然后在CD命令中执行Helm升级命令即可。
即使有问题，也可以一次性回滚Chart内的所有资源，而不是一个个回滚。可见，Helm大大提高了部署和回滚的效率。

如果你想要进一步了解Helm，可以参考我的另一篇文章[_Helm手记_](doc_helm.md)。

### 5.9 一些建议

CI/CD流水线配置很难一开始就做到完美，但可以参考下面原则来尽可能完善流程：

- 在CI中关注自动化和构建速度。优化构建速度能够在代码提交时快速完成构建，为开发者提供更快速的反馈。
- 在流水线中配置可靠且完善的测试脚本。以便在代码提交时，能够快速找到代码中的问题，以便快速修复和验证。
- 尽可能优化镜像大小，这样可以减少拉取镜像的时间，提高构建速度，还能减少容器运行时的内存消耗和受攻击面。
- 如果还不熟悉CD，可以先从简单易用的滚动更新开始。后续再考虑使用金丝雀发布或蓝绿部署。
- 在CD环节，一定要注意对客户端连接的重新建立和数据库结构的变更进行充分测试，尽可能减少对用户端的影响。
- 生产环境中进行测试时，应该从小规模开始，逐步扩大测试范围。

## 6. 资源管理

当应用部署到Kubernetes集群中以后，我们的工作还没有结束，因为实战中总是会出现各种各样的突发状况，
比如，某个服务突然变得非常慢，某个服务突然变得不可用，某个服务需要增加实例数量，某个服务需要减少实例数量等等。
为了应对这些突发状况，我们需要使用Kubernetes内置的一些资源管理功能。

具体来说，本节将会介绍以下Kubernetes功能：

- Pod调度原理
- Pod调度技术
    - Pod的亲和与反亲和
    - 节点的亲和与反亲和
    - 节点选择器
    - 污点和容忍度
    - 指定节点名称
- Pod资源管理
    - Pod资源请求和限制
    - Pod服务质量
    - PodDisruptionBudget
    - Namespace维度的资源管理
        - ResourceQuota
        - LimitRange
- 资源伸缩
    - Pod水平伸缩
    - Pod垂直伸缩
    - 节点水平伸缩

本文会对这些功能进行一个大致的介绍以及给出相应的参考🔗，但介于篇幅不会对每个功能都进行详细的使用介绍。

### 6.1 Pod调度原理

当一个Pod被提交到Kubernetes集群中以后，`kube-schduler`会负责决定Pod最终运行在哪个节点。
具体来说，`kube-schduler`的调度算法大致分为**预选、优选和绑定**三个阶段。

**预选阶段**主要是通过一些内置规则来过滤掉不符合需求的节点。比如，节点资源是否满足Pod的资源请求和限制，
节点是否满足Pod的亲和与反亲和规则等等。具体的内置规则介绍可以查看[
_Kubernetes进阶教程-预选阶段_](doc_tutorial_senior.md#411-预选阶段)。

**优选阶段**则是在预选阶段选出的节点列表中进一步对这些节点进行打分，当然也是根据一些内置规则。比如节点空闲资源多的得分更高、未运行相同应用Pod的节点得分更高等等。
具体的内置规则介绍可以查看[_Kubernetes进阶教程-优选阶段_](doc_tutorial_senior.md#412-优选阶段)。

**绑定阶段**是选择分值最高的节点作为Pod运行的目标节点进行绑定（如有多个，则随机一个）。

### 6.2 Pod调度技术—Pod的亲和与反亲和

Pod的亲和与反亲和是一项可供手动配置的Pod资源调度技术，它属于前面提到的**优选阶段**的一种规则。
我们可以通过设置它来决定如何部署相关联的一组Pod。

例如，为Pod设置亲和性规则可以让Pod仅运行或尽可能运行在满足条件（这些节点运行了某些偏好Pod）的节点上，理由可能是为了集中管理某些Pod。
同理，Pod的反亲和性规则是让Pod必须或尽可能不运行在满足条件的节点上，理由可能是远离运行了某些Pod的节点。

如果你想进一步了解此项技术的使用配置方法，可以参考[
_Kubernetes进阶教程-Pod亲和性和反亲和性_](doc_tutorial_senior.md#45-软硬皆可-pod亲和性和反亲和性)。

### 6.3 Pod调度技术—Node的亲和与反亲和

Node的亲和与反亲和与Pod的亲和与反亲和类似，只是它的配置目标针对的是节点而不是Pod，
请直接参考[_Kubernetes进阶教程-节点亲和性_](doc_tutorial_senior.md#44-软硬皆可-节点亲和性affinity)。

### 6.4 Pod调度技术—节点选择器

nodeSelector是将Pod调度到特定节点最简单的方法。使用步骤是先为目标节点设置标签，然后在PodSpec中设置nodeSelector字段。
需要注意的是，nodeSelector是一项硬性调度配置，若没有满足标签要求的节点，则Pod将无法调度到任何节点上（处于Pending状态）。
例如，你可能希望将Pod调度到具有特定硬件配置（如GPU）的节点上。

- [_Kubernetes进阶教程-指定节点标签_](doc_tutorial_senior.md#42-硬性调度-指定节点标签nodeselector)

### 6.5 Pod调度技术—污点与容忍度

污点与节点绑定，节点拥有污点后防止Pod被调度到该节点，除非PodSpec中配置了容忍度（针对该污点）。
每个节点都可以设置污点（taint），污点是一个键值对（值可忽略）和污点效果的形式，比如`role/log:NoSchedule`。

污点与反亲和性类似都是用来排斥Pod，但它们之间有一个重要区别，那就是污点默认排斥所有的Pod（直到为Pod设置了对应污点的容忍度）。
例如，在上线某些GPU节点时，就可以将它们设置污点，这样Pod就不会被调度到这些节点上，直到你手动为那些需要GPU的Pod设置容忍度。
还有就是可以用来排空节点，比如现在想要下线或维护某个节点，就可以为其设置一个污点叫做`maintaining:NoExecute`，
在短暂时间后，Kubernetes就会排空该节点上的Pod（强制排空），当Pod被排空后，就可以将节点下线或进行维护。

- [_Kubernetes进阶教程-污点和容忍度_](doc_tutorial_senior.md#46-污点和容忍度)

### 6.6 Pod调度技术—指定节点名称

通过在PodSpec中指定`nodeName`字段，可以让Pod直接调度到指定节点上，并且可以自动容忍节点上的除了包含`NoExecute`影响以外的污点。
这个特性由于灵活性较低所以使用较少。使用时需注意，若节点名称不存在于集群的节点列表中，则Pod将无法调度到任何节点上，通过`kubectl get po`
也无法看到Pod。

- [_Kubernetes进阶教程-指定节点名称nodename_](doc_tutorial_senior.md#43-硬性调度-指定节点名称nodename)

### 6.7 Pod资源管理—请求和限制

在实践中，一般都需要在PodSpec中设置容器的`requests`和`limits`字段，分别表示Pod内容器对CPU和内存的最低需求和最高限制，
前者可以保证Pod能够正常调度到资源足够的节点上运行，后者是为了避免Pod占用过多节点资源影响到节点上的其他应用。当Pod对资源的占用达到限制后，
Pod可能因为OOM而重启或调度到其他节点。

当你安装了MetricsServer后，就可以通过`kubectl top nodes`命令查看所有节点的资源使用情况，如下所示。

```shell
# CPU资源是以millicore（m）为单位，1000m=1core
# 内存资源是以byte为单位，1Mi=1024Ki=1024*1024bytes
$ kubectl top nodes
NAME                      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
test-1.27-control-plane   241m         6%     559Mi           14%       
test-1.27-worker          105m         2%     742Mi           18%       
test-1.27-worker2         93m          2%     1788Mi          45%   
```

根据这些信息我们可以知道所有节点的CPU和内存的已使用情况，从而知道何时该为集群增加节点。同时也可以使用`kubectl top po`
查看Pod对节点资源的使用情况。

- [_Kubernetes进阶教程-安装metrics-server插件_](doc_tutorial_senior.md#341-安装metrics-server插件)

### 6.8 Pod资源管理—服务质量（QoS）

Pod被创建后会被分配一个QoS等级，当节点资源不足时，会根据Pod的QoS等级作为依据来驱逐某些Pod。
具体分配的QoS等级由Pod的资源请求和限制决定。一共有以下3种QoS等级：

- Guaranteed：当Pod中每个容器（包含初始化容器和普通容器）都定义了CPU或内存的`requests`和`limits`且数额相等时，分配此等级。
    - 最高优先级，当节点资源不足时，系统会选择Kill其他两种优先级的Pod来保证此等级Pod的正常运行。
- Burstable：当Pod中存在一个容器的CPU或内存的`requests`小于`limits`时，分配此等级。
    - 次高优先级，当节点资源不足时，系统会选择Kill BestEffort等级的Pod来保证此等级Pod的正常运行。
- BestEffort：当Pod中没有任何容器设置对CPU或内存的`requests`和`limits`时，分配此等级。
    - 最低优先级，当节点资源不足时，系统会优先Kill此等级的Pod。

注意，若某个容器只设置了CPU或内存的`limits`字段但未设置`requests`，Kubernetes会自动为其设置与限制相等数额的请求。

查看Pod服务质量等级的命令：

```shell
kubectl get po xxx -o jsonpath='{ .status.qosClass}{"\n"}'
```

所以，为了避免Pod被Kill，应该尽可能为重要业务的Pod设置Guaranteed的QoS等级。

### 6.9 Pod资源管理—PodDisruptionBudget

正常运行的Pod可能在某个时候被人工或控制器驱逐（相当于删除），驱逐原因分为两类：主动
驱逐和被动驱逐。主动驱逐是指用户或控制器明确地执行了删除操作（包括对Pod模板的更新、排空节点等）；
而被动驱逐是指由于节点资源不足，Kubernetes系统根据QoS等级选择性地驱逐Pod，被动驱逐还包括硬件故障、网络分区或内核崩溃等异常情况。

任何原因导致的驱逐都可以成为干扰（Disruption），为了最大程度减少干扰对应用的影响，
Kubernetes允许我们设置PDB来保证发生干扰时应用仍然能够正常运行。具体来说，PDB可以设置在发生干扰时，
与指定标签匹配的Pod最多可以被关闭的数量或百分比、最少可用的Pod数量或百分比。

当进行排空（drain）操作时，如果违反PDB策略，排空命令将进入阻塞。

> 注意：工作负载资源（Deployment、StatefulSet或ReplicaSet等）在滚动升级时不会触发PDB。

PDB的示例模板：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  # minAvailable 和 maxUnavailable 只能设置其一，两者都支持整数和百分比
  minAvailable: 2
  # maxUnavailable: 10%
  selector:
    matchLabels:
      app: zookeeper
```

PDB根据selector匹配一组Pod，它们可以是某个控制器（如Deployment或StatefulSet等）下的Pod，也可以是独立的Pod。

- [为应用程序设置干扰预算](https://kubernetes.io/zh-cn/docs/tasks/run-application/configure-pdb/)

### 6.10 Pod资源管理—ResourceQuota

当多个团队或多个不同类别的应用共享一个集群时，我们可能需要将它们安置在不同的命名空间下进行管理。进一步，
我们还需要对每个命名空间的资源额度进行限制，否则就会互相影响，导致意外结果。

ResourceQuota是用于管理单个命名空间的总资源限额的Kubernetes API对象。它可以为以下资源进行限额：

- CPU/内存的请求和限制
- 存储卷总量
- PVC个数
- Pod/Service/Deployment/ReplicaSet等资源的个数

当命名空间中的资源使用量达到限额就会触发错误提示。

- [_Kubernetes进阶教程-配置整体资源配额_](doc_tutorial_senior.md#321-配置整体资源配额)

### 6.11 Pod资源管理—LimitRange

ResourceQuota是管理单个命名空间的总资源限额，但我们还需要对命名空间下的单个对象所使用的资源进行限额，否则在命名空间内的对象也会互相影响。
而LimitRange则可以帮助我们达成目标，例如，它可以设置命名空间下Pod的默认CPU/内存的请求和限制的数值，当Pod的设置超出限制时无法部署，
同时当LimitRange配置后，新的（包括控制下的）Pod必须设置CPU/内存的请求和限制，否则也无法部署。

除了Pod，LimitRange还可以针对容器、PVC等许多对象进行配置。

- [_Kubernetes进阶教程-配置个体资源配额_](doc_tutorial_senior.md#323-配置个体资源配额)

### 6.12 资源伸缩—Pod水平伸缩

当Kubernetes中的应用突然面临业务高峰期，固定副本数量部署的Pod无法满足业务需求时，
我们可以通过Horizontal Pod Autoscaler（HPA）对Pod进行水平伸缩。
HPA是Kubernetes提供的对象，可以帮助我们根据Pod的CPU/内存使用率自动调整Pod的副本数量，从而实现Pod副本数量的自动伸缩。

除了使用Pod的CPU/内存使用率作为伸缩指标，还支持使用自定义指标，但这需要额外部署资源来作为指标来源。

- [_Kubernetes进阶教程-使用HPA水平扩缩Pod_](doc_tutorial_senior.md#34-使用hpa水平扩缩pod)

### 6.12 资源伸缩—Pod垂直伸缩

HPA是一种简单直接的增加服务端吞吐量的方式，但如果Pod由多个容器组成，而只有一个接收流量的应用容器，HPA可能会造成一些资源浪费。
此时，使用Vertical Pod Autoscaler（VPA）可以实现Pod的垂直伸缩。
VPA是Kubernetes提供的对象，可以帮助我们根据Pod中所有容器的CPU/内存使用率自动调整Pod中所有容器的CPU/内存的请求和限制的数值，
从而实现Pod中所有容器的CPU/内存的自动伸缩。

VPA并不是Kubernetes原生提供的功能，而是由社区提供的，需要以自定义资源的方式进行安装。VPA提供四种工作模式：

- auto：VPA在Pod创建时为其分配资源请求，并在Pod运行时根据Pod的资源使用率自动调整资源请求。
    - 当前的更新是通过删除重建（等效于`recreate`）的方式，一旦Kubernetes支持原地更新，VPA将切换到原地更新模式。
- recreate：VPA在Pod创建时为其分配资源请求，并在Pod运行时根据Pod的CPU/内存使用率自动调整资源请求，更新是以删除重建的方式进行。
- initial：VPA只在创建Pod时分配资源请求，以后不会更改。
- off：VPA不会自动更改Pod的资源请求。

下面是一个VPA配置实例（安装VPA后可用）：

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
```

VPA的使用比较复杂，有许多需要注意的点。例如，VPA不建议与HPA同时使用（除非HPA使用自定义指标作为参考）。若要使用VPA，请阅读VPA文档，
并进行详尽的测试后再投入生产。

- [VPA使用介绍](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)

### 6.13 资源伸缩—节点水平伸缩

当面对具有更大突发流量的业务时，考虑实施节点维度的自动水平伸缩是一个不错的选择。
但目前的集群自动伸缩功能**仅支持在公有云上**运行，这点需要注意。

节点的水平伸缩会在以下两种情况自动触发：

- Pod由于资源不足而无法启动时（增加节点）。
- 集群中存在已经在较长的时间段内未被充分利用的节点，并且它们的Pod可以被放置在其它现有节点上（缩减节点）。

节点水平伸缩是一个高级的主题，考虑在合适的时机去增减节点（以及增减哪些节点）是一个十分复杂的决策过程，
这其中需要考虑到Pod亲和性和反亲和性、污点、运行PDB的Pod的节点等等。若你直接使用云托管的K8s集群，
则可以直接使用云厂商原生支持的集群节点自动伸缩功能，而无需自行运维此功能。

- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)

## 7. 网络策略与Pod安全

### 7.1 网络策略

默认情况下，集群中的工作负载（Deployment等控制器下的Pod）资源的网络通信是开放的，可以随意访问。
这包括（可能跨命名空间的）Pod之间和Pod与外部世界之间的互通性。
开放访问虽然减少了运维复杂度和困扰，但同时也带来了风险。例如，DB服务应该只允许被部分运行后端服务的Pod访问，
那些前端应用Pod和外部世界绝不应该访问到DB服务。此时，应该有一条ACL（访问控制列表，传统网络层面的术语）来实现此目标。

好在，Kubernetes提供了一个叫做**NetworkPolicy**的API资源来为集群的网络层（第四层）保驾护航。
NetworkPolicy可以被看做是集群中的一个东西向流量防火墙，每个策略规则都通过`podSelector`属性来匹配一组Pod，同时控制它们的流量出入。
每条网络策略都应用于Pod流量的一个或两个方向，即`Egress`（出站方向）和`Ingress`（入站方向），
每个方向指定目的地或来源时都可以选择三种方式限定：匹配某些标签的一组Pod、一个IP块或匹配某些标签的命名空间，它们之间可以同时指定，是或的关系。
同时还可以指定哪些端口和协议（支持TCP/UDP）可以被访问，或作为目的地。

注意几点：

- 每个方向都可以指定多个目的地或来源，这些目的地或来源之间是或的关系。但进一步，每个目的地或来源又可以由上面提到的三种方式进行任意组合，组合后的结果是且的关系，具体请参阅官文。
    - 简单的总结就是外或内且。
- 对于除了TCP/UDP以外的协议的过滤支持，取决于集群所安装的CNI网络插件。
- 对于使用`hostNetwork`的Pod，NetworkPolicy无法保证对其生效。
    - 这类Pod对外访问时具有与节点相同的IP，可以使用`ipBlock`规则允许来自`hostNetwork`Pod的流量。
- 不管是出站还是入站流量策略，都是叠加生效的，最终效果不受顺序影响。
    - 由此推断，较多的网络策略会影响集群网络性能。
- 新增策略对现有连接的影响是不确定的，具体行为由CNI网络插件决定。
    - 例如，旧策略是允许来自某个源的访问，应用的新策略则是拒绝这个源的访问，此时是立即切断现有连接还是仅对新连接生效
      取决于CNI网络插件的实现。
- 支持NetworkPolicy的CNI网络插件有：Calico、Antrea、Cilium、Kube-router、Romana、Weave Net等。注意，Flannel不支持。

#### 7.1.1 出站流量

当没有`Egress`策略时，它将默认允许Pod的所有出站流量。当存在一条`Egress`策略时：

- 它将控制（且仅）一组通过`podSelector`匹配的Pod的出站流量；
- 它将允许匹配的一组Pod**访问**指定目标的流量，同时默认允许这个目标网络的应答流量（不需要Ingress规则允许）；
- 若策略没有指定出站目标，则表示不允许任何出站流量；
- 所匹配的这组Pod访问任何非本地、且非目标的流量都将被拒绝，除非有其他Egress策略放行。
- 若策略没有指定`podSelector`（该字段留空），则表示策略所在命名空间下的所有Pod都将应用此策略，通常用于**默认拒绝出站**规则。

#### 7.1.2 入站流量

当没有`Ingress`规则时，它将默认允许Pod的所有入站流量。当存在一条`Ingress`策略时：

- 它将控制（且仅）一组通过`podSelector`匹配的Pod的入站流量；
- 它将允许一组匹配的Pod**接收**指定的来源网络的流量，同时默认允许流向这个来源网络的应答流量（不需要Egress规则允许）；
- 若策略没有指定入站来源，则表示不允许任何入站流量；
- 所匹配的这组Pod不会收到任何非本地、且非指定来源的流量（被网络插件拒绝），除非有其他Ingress策略放行。
- 若策略没有指定`podSelector`，则表示策略所在命名空间下的所有Pod都将应用此策略，通常用于**默认拒绝入站**规则。

注意，要允许从源 Pod 到目的 Pod 的某个连接，源 Pod 的出口策略和目的 Pod 的入口策略都需要允许此连接，
除非流量源没有应用任何出站策略或流量目的地没有应用任何入站策略。

#### 7.1.3 示例

示例模板：[network-policy.yaml](k8s_actions_guide/version1/other_manifest/network-policy.yaml)

模板将实现以下目标：

- 默认拒绝default命名空间下的所有Pod的入站流量；
- 允许default命名空间下携带标签`access-db: mysql`的Pod访问MySQL服务（携带标签`db: mysql`），策略应用于DB侧的入站流量；
- 拒绝default命名空间下携带标签`internal: true`的Pod的出站流量；
- 允许default命名空间下携带标签`internal: true`的Pod访问`20.2.0.0/16`网段且目标端口是5978的TCP端口。

> 注意：
> 该示例没有完全展示出`Ingress`和`Egress`的规则组合，
> 更完整的示例请参考官方的
> [NetworkPolicy](https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/#networkpolicy-resource)
> 文档。

### 7.2 Pod安全

#### 7.2.1 Pod安全策略

虽然K8s拥有RBAC的权限控制机制，但RBAC仅能限制用户对K8s API对象（如Pod/Deployment/Service等）的访问，
无法限制Pod内部的安全权限（例如拥有特权/访问宿主机文件系统/特权提升/使用hostNetwork/设置seLinux等）的使用。
一个不安全的镜像，或者一个配置了特权权限的容器，都可能导致集群的安全问题。

因此，早在v1.3版本中，K8s就提供了 **PodSecurityPolicy**（PSP） 这项API对象来为Pod设置安全策略。
PSP可以防止Pod未经授权的获取节点root权限或访问节点的其他敏感资源（如网络/文件系统等），从而影响集群的安全。

**PodSecurityPolicy的废弃**

- 由于PSP在使用上的繁琐以及部分限制，在v1.21版本中，PSP被标记为`Deprecated`，并在v1.25版本中正式去除。
- [弃用 PodSecurityPolicy：过去、现在、未来](https://kubernetes.io/zh-cn/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)

所以，本文不会介绍PSP的使用，感兴趣的读者可以查阅官方文档。

#### 7.2.2 Pod安全准入控制器

弃用PSP后，K8s官方重新构思并实现了一套新的Pod安全控制机制叫做Pod安全准入控制器（Pod Security Admission Controller），
这套机制致力于在保留PSP大部分功能的同时提升了使用上的简便性，最大的特点是支持柔性上线能力。

Pod安全准入控制器是K8s v1.22版本引入的一项功能（此版本需要修改API-Server的启动参数进行手动开启），在v1.23版本中作为Beta特性默认开启。
不过，这项功能并不像PSP那样通过API对象来提供能力，而是通过K8s原有的准入控制器（Admission Controller）来提供能力。
具体来说，新的功能首先提出了一个Pod安全性标准为Pod定义了不同的隔离级别，这些标准能够让你以一种清晰、一致的方式定义 Pod
的安全行为。其次，新功能增加了一个叫做PodSecurity的准入控制器，它将在命名空间维度来执行这个Pod安全标准。

Pod安全标准定义了三个隔离级别：**privileged**、**baseline** 和 **restricted**，从左到右分别是特权/基准/严格，限制程度从低到高。
我们需要先为命名空间设置一个安全标准，以及一个模式表示违反标准时该如何处理，可以设置多个标签，且安全标准和模式可以任意组合。
一共有三个模式可用：enforce/audit/warn，其中enforce是强制模式（拒绝违反安全标准的Pod），audit是审计模式，warn是警告模式。

下面的清单定义了一个`my-baseline-namespace`命名空间，声明了如下限制：

- 阻止任何不满足 baseline 策略要求的 Pod；
- 针对任何无法满足 restricted 策略要求的、已创建的 Pod 为用户生成警告信息， 并添加审计注解；
- 将 baseline 和 restricted 策略的版本锁定到 v1.29。

```yaml
# 本模板来自官网示例
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    # 形式：pod-security.kubernetes.io/<MODE>: <LEVEL>
    pod-security.kubernetes.io/enforce: baseline
    # 形式：pod-security.kubernetes.io/<MODE>-version: <VERSION>
    # 注：这个标签表示此模式按照指定版本的规则进行检查，如果未指定，则按当前K8s版本
    #  - 例如，v1.24 及之前版本的 Kubelet 不强制检查 PodSpec中的 OS 字段，若你也不希望检查此字段，则需要锁定到v1.24 及之前的版本
    #  - （但如果当前版本比锁定版本还低，则使用当前版本，即使用二者之间较低的版本对应的检查规则）
    pod-security.kubernetes.io/enforce-version: v1.29
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.29
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.29
```

在应用这个清单后，准入控制器会检查当前命名空间中存在的Pod，任何违反 enforce 策略要求的 Pod
都会通过警告的形式提示。对于已存在的命名空间，可以通过 `kubectl label`
命令进行修改。

为命名空间设置好安全标准标签后，接下来就是**按照所设置的安全标准来填写/纠正Pod模板**。不同的安全标准级别对PodSpec中字段的要求不同，
`privileged`是特权级别没有限制；而`baseline`和 `restricted`则存在诸多限制。
具体要求在 [Pod安全性标准—Profile](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/#profile-details)
页面中可以找到。

例如，按照上面的清单创建的`my-baseline-namespace`命名空间，在创建Pod时，需要按照`baseline`安全标准来填写PodSpec。
部署下面的Pod清单将会被拒绝，并得到提示：`Error from server (Forbidden): error when creating "pod_curl.yaml": pods "curl" is forbidden: violates PodSecurity "baseline:v1.29": host namespaces (hostNetwork=true)
`。

```shell
# pod_curl.yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
  namespace: my-baseline-namespace
spec:
  hostNetwork: true
  containers:
    - name: curl-container
      image: appropriate/curl
      command: [ "sh","-c", "sleep 1h" ]
```

需要注意的是，`enforce`模式**仅对原生Pod生效**，不会对Pod控制器（如Deployment）生效。其他两种模式则是对控制器和原生Pod都生效。

- [Pod安全准入](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission/)
- [Pod安全性标准](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards)
- [使用名字空间标签来实施 Pod 安全性标准](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)

#### 7.2.3 白名单

Pod安全准入控制器会对命名空间下的所有Pod或控制器的 PodSpec 进行安全规则校验，
如果 PodSpec 中存在不安全的字段设置，则拒绝创建Pod或给出提示。但凡事总有例外，Kubernetes就允许在准入控制器中静态配置这些例外，
这里我们称作白名单，官方称作豁免（Exemptions）。

白名单可以通过三种方式配置：

- Username：免去对指定用户创建的原生Pod的安全检查，控制器除外。
- RuntimeClassName：免去对包含指定运行时类的Pod或控制器的安全检查。
- Namespace：免去对指定命名空间下所有Pod或控制器的安全检查。

具体配置方式并不复杂，请参阅[官网文档](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller)。

## 8. 服务网格

本节是一个笔者新增的一个大章节，严格来说本节主题与K8s并不绑定，但笔者认为服务网格是在K8s中运行微服务架构的一个流行的发展方向（其实在大厂或国内早已普遍应用）。
可能由于早期的文档或教程缺失，导致国内开发者对服务网格的认知和实践步骤并不清晰，因此在本节中，笔者将结合
一些权威书籍、官方指导以及笔者在实际生产环境中的实践经验，对服务网格进行一个系统的理论和实践介绍。
此外，本章节也将作为一个特别篇在项目主页中进行展示。

> 此章节中的演示将继续复用第一节中的demo项目。

### 8.1 出现时间与背景

早在十多年前，国外就开始流行起了从单体服务转向微服务架构的潮流，并在2014年传入国内。
微服务架构的优点也是有目共睹的，例如松耦合、快速迭代、高可扩展性、支持语言异构等。也正是微服务架构带火了云原生概念，
因为微服务的诸多优点都能够在云原生环境下得到完美体现。

然而，想要在企业中部署微服务架构也不是那么容易。由于微服务架构的多服务以及单服务多副本等特性，
保证服务间的正常、高效且安全的通信是一个需要密切关注的问题。传统微服务架构中，
我们需要为每个服务配置服务发现、负载均衡、服务路由、服务监控、服务容错等基础设施。当然，
我们可以为相同语言实现的微服务配置相同的一系列基础设施，但一旦出现其他语言构建的服务，
则又要单独去添加这些基础设施，这就造成了重复工作，还带来了大量的排错及维护成本。

上面提到的一系列基础设施，它们可能是由不同的第三方包提供（也叫做基于SDK的架构模式），
这种独立分散的方式也带来了服务组件的版本管理、依赖管理、服务部署、服务监控等问题的困扰。
并且使用它们的逻辑一定程度上耦合在业务代码中，这也使得整个代码库变得臃肿且增加了复杂度。

**服务网格的诞生**

为了解决上述问题，服务网格应运而生。服务网格是一个基础设施层，它位于应用程序和基础架构之间，
为应用程序提供服务间通信功能。服务网格由一系列轻量级网络代理组成，这些代理部署于每个应用实例旁（以sidecar模式），
并负责处理服务间的通信，包括服务发现、负载均衡、服务路由、服务监控、服务容错等。从此，
业务代码库中不再需要包含涉及处理服务间网络通信的逻辑，团队成员的编程工作又回归到单体服务架构的开发模式中，
即只需要专注于业务逻辑的开发。

世界上首个Service Mesh产品是国外的Buoyant公司在2017年4月发布的Linkerd。
同时，Service Mesh概念也是由这家公司的CEO William
Morgan提出，他同年发表的文章 [What’s a service mesh？And why do I need one?][What’s a service mesh？And why do I need one?]
是对Service Mesh概念公认最权威的定义。

之所以称为服务网格，是因为有了代理之后，所有服务间的通信都变成代理间的通信，而这些代理间的通信又形成一个巨大的网格，
并且整张网格都由一个统一的控制平面来进行管理。这可以通过下面的图加深理解

<div align="center">
<img src="img/service-mesh.jpg" width = "450" height = "400" alt=""/>
</div>

### 8.2 定义

"Service Mesh 是一个处理服务通讯的专门的基础设施层。它的职责是在由云原生应用组成服务的复杂拓扑结构下进行可靠的请求传送。
在实践中，它是一组和应用服务部署在一起的轻量级的网络代理，对应用服务透明。" —— William Morgan

具体来说，Service Mesh使用的网络代理本质上是一个容器，它以sidecar模式部署在每个微服务侧（对应用实例完全透明），
并且它会接管同侧的应用实例发出和接收的流量，根据配置规则，
它可以对流量进行重定向、路由、负载均衡、监控、熔断等原来需要由多个工具完成的操作。
Service Mesh将服务通信及相关管控功能从业务程序中分离并下层到基础设施层，使其和业务系统完全解耦。

### 8.3 控制平面

最初的Service Mesh产品形态中，并不存在统一的控制平面，每个代理的配置规则的变更都需要单独手动操作，这就显得十分麻烦。
但很快（也是2017年），以Istio为代表的第二代Service Mesh产品形态出现了，它们都具备一个统一控制平面，代理则称为数据平面。
通过控制平面，我们可以集中管理所有代理的配置规则（还包括代理的数据采集等功能），
而数据平面的每个代理还负责采集和上报网格流量的观测数据。

### 8.4 Istio

虽然到目前为止，在CNCF旗下托管的Service Mesh生态圈已经呈现繁荣姿态，例如Linkerd、Istio、Consul Connect、Kuma、Gloo Mesh等。
但由于庞大的贡献者数量和社区支持，最终**Istio成为服务网格领域的领先者**。能够与之比较的是商业产品Linkerd，它的优势是更轻量，
适合部署在中小规模云环境中，但缺少部分Istio才有的高级功能。关于Istio与Linkerd的详细对比，请查看
[Istio vs Linkerd: The Best Service Mesh for 2023][Istio vs Linkerd]。

Istio最初由Google、IBM和Lyft等公司共同开发。在2018年发布了其1.0版本。随后，Istio在2022年4月宣布捐赠给CNCF（正式孵化时间是同年9月底），
最终Istio在2023年7月正式毕业（不到一年），且如今已经有 **数百家** 公司为其贡献代码。
截至当前（2024年3月5日），它已迭代至v1.20，可见其发展之迅速。

**Istio的口号**

“Simplify observability, traffic management, security, and policy with the leading service mesh.”

使用领先的服务网格技术来简化可观测性（WebUI、4/7层流量指标、延迟、链路跟踪、健康检查、日志、网络拓扑）、
流量管理（负载均衡、多集群流量路由、服务发现、熔断/重试/超时/限速、动态配置、HTTP 1.1/2/3支持、TCP/UDP/gRPC支持、延迟注入等）、
安全（认证、mTLS等）和策略。

> Istio目前在中国互联网公司中并没有大量普及，这主要是由于其上手成本较高，在生产环境中应用Istio需要团队中至少有一位Istio
> 专家或有丰富实践经验之人，而且最好对Istio的原理有深入的理解，这样
> 才能在出现问题时不会手忙脚乱。此外，根据个人经验，
> 如果你环境中的微服务数量低于10个，不建议使用Istio，因为它的上手和维护成本可能会超过其带来的收益。

#### 8.4.1 基本架构

部署后的Istio架构包含两个部分：

- **数据平面**：又叫数据层，由N个代理服务（每个代理都是一个叫`istio-proxy`的sidecar容器，它基于Envoy）组成，
  负责网格中的服务通信和流量管控，同时会收集和上报网格流量相关的观测数据。
    - 早期通过人工的方式将Envoy容器硬编码到每个工作负载模板中，后期发展至可以通过Istio的sidecar注入功能在部署工作负载时自动向模板注入Istio容器（通过K8s提供的Webhook机制），
      大大提高使用效率，避免了对业务负载模板的入侵。
    - 代理服务会劫持应用容器发出和接收的流量，然后按配置进行相应的处理，最后再决定是丢弃还是将流量转发出去。
    - 向应用Pod注入sidecar容器时，同时还会注入一个`istio-init`**初始化**容器，负责设置Pod的iptables规则，以便入站/出站流量通过
      sidecar 代理转发。
    - Istio会跟踪K8s集群中的Services和Endpoints的变化，进而下发到每个Envoy代理中，让其可以知晓转发的实际目的地址。
- **控制平面**：又叫控制层，由一个叫做`istiod`的服务组成，负责整个数据平面的配置规则（运行时）下发和观测数据的管理。同时支持身份标识和证书管理。
    - istiod内部有三大组件：Pilot、Citadel、Galley
        - Pilot负责收集和分发网格中的服务配置信息，包括服务发现、负载均衡、路由规则、访问控制等。
        - Citadel负责为网格中的服务提供身份标识和证书管理，包括服务证书的签发、过期提醒、密钥轮换等。
        - Galley负责对用户提供的配置进行格式以及内容验证、转换和下发，它直接与控制面内的其他组件通信，主要负责将控制面的其他组件与底层平台（如K8s）解耦。

Istio架构图如下：

<div align="center">
<img src="img/istio-architecture.png" width = "800" height = 600" alt=""/>
</div>


**独立的Envoy**  
需要注意的是，Envoy组件是可以单独运行的，就像一个Nginx进程一样，只要你为它添加合理的配置。

**部署模型**

Istio支持的部署模型比较繁杂，它可以根据多个维度进行分类，包含集群模型/网络模型/控制平面模型/网格模型/租户模型。
任何一个包含Istio的生产环境都涉及这几个维度。当然最简单的部署模型就就是单一集群+单一网络+单一控制平面+单一网格+单一租户。
当部署Istio时，我们需要根据服务规模和团队数量来规划具体的部署模型。

> 请参考 [部署模型](https://istio.io/latest/zh/docs/ops/deployment/deployment-models/) 来详细了解Istio的部署模型。


**更多细节**

- [Envoy介绍](https://www.thebyte.com.cn/MicroService/Envoy.html)
- [Istio介绍](https://www.thebyte.com.cn/MicroService/Istio.html)
- [sidecar自动注入原理](https://istio.io/latest/zh/blog/2019/data-plane-setup/#automatic-injection)
- [sidecar流量捕获原理](https://istio.io/latest/zh/blog/2019/data-plane-setup/#traffic-flow-from-application-container-to-sidecar-proxy)

#### 8.4.2 Ambient Mesh

**sidecar模式的弊病**

不管是第一代还是第二代Service Mesh技术，都存在着一个令人诟病但不易解决的难题，
那就是sidecar模式带来的资源消耗和通信时延问题。根据统计，大部分企业在使用Istio的数据平面即Envoy时的平均内存开销都在60M~
100M左右，
如果是稍微大一点的规模比如100个服务，那么sidecar这部分开销就可能接近10个G，并且由于每个代理都会接收所有的服务发现数据（即使不需要），
这会导致代理的内存开销会随着服务规模的增长而呈指数级增长。此外，还有sidecar带来的网络多跳和路由计算所增加的网络延迟问题，
平均延迟大致为3ms~5ms左右，这对于延迟敏感的业务中可能不会适用。

**Istio的新数据平面模式：No sidecar的Ambient Mesh**

2022年9月，Istio官宣了一种新的数据平面模式——Ambient Mesh。它能够在简化操作、保持更广泛的应用程序兼容性和降低基础设施成本的同时，
保持Istio的零信任安全、遥测和流量管理等核心功能。最重要的是，它摒弃了传统的sidecar部署模式，
转而在每个节点上部署一个零信任的**ztunnel**代理用来为节点上所有Pod提供mTLS、遥测、身份验证和L4授权，它并不会解析HTTP流量（L7）。

这种新的平面模式将Istio的功能划分为两层：

- 基础层：作为一个必需的安全覆盖层，处理所有应用实例的路由和零信任安全的流量。这一层产生的性能开销较低；
    - 具体来说，Istio会在每个节点上部署一个代理（叫做ztunnel），用来处理该节点上的所有L4流量（如TCP、UDP等）；
- 应用层：使用一个七层代理来负责处理七层流量，如HTTP、gRPC等。这一层产生的性能开销大于基础层；
    - 具体来说，Istio会在集群中新增一个命名空间来部署基于Envoy的Waypoint代理（以Pod形式），由它来处理和转发**需要**
      执行七层流量策略的实例流量；
    - Istio的控制平面会负责Waypoint代理策略的下发；
    - Waypoint代理的数量可以根据集群流量规模实现自动缩放。

用户在部署此模式时可能并不需要安装“应用层”，即不需要在实例之间处理七层流量。这样一来就可以按需使用Istio提供的功能，
相较sidecar模式的无差别全功能提供而言，Ambient Mesh模式大大减少了Mesh架构产生的开销。

**尚未准备好用于生产环境**

截至目前（2024/3/6），Ambient Mesh是仍处于Alpha阶段的特性，比预期进度（原本预计2023年底Prod Ready）要慢，相关文档还在持续更新中，请持续观望。

#### 8.4.3 安装过程

##### 8.4.3.1 安装

当你准备好开始学习Istio时，笔者建议你使用具备高度定制化功能的`istioctl`命令行工具进行安装，在熟悉安装流程后，再尝试使用Helm或其他方式进行安装。
完整的安装指南请参考[Istio安装指南][Istio安装指南]。

> 注意，Istio原生与Kubernetes集成（但不绑定），安装Istio时会将一些自定义资源安装到Kubernetes集群中，因此请提前准备好一个集群以供安装。

安装步骤：

```shell
# 1. 在istio发版页面复制需要安装的`istioctl`链接（注意不是istio，以及选择符合节点架构的版本，下面以v1.20.3和linux amd64架构为例）
# https://github.com/istio/istio/releases

# 进入任何一个具有K8s集群管理员权限的主机
$ wget https://hub.gitmirror.com/?q=https://github.com/istio/istio/releases/download/1.20.3/istioctl-1.20.3-linux-amd64.tar.gz -O istioctl.tar

# 解压
$ tar -zxf istioctl.tar && chmod +x istioctl

# 生成即将安装到集群的istio资源的k8s清单文件，稍后用于验证安装是否成功
$ ./istioctl manifest generate > manifest.yaml

# 使用默认配置进行安装
# - 此过程会安装Pod，所以需要拉取远程镜像，可能需要几分钟
$ ./istioctl install
This will install the Istio 1.20.3 "default" profile (with components: Istio core, Istiod, and Ingress gateways) into the cluster. Proceed? (y/N) y
✔ Istio core installed                                                                                                                                                                    
✔ Istiod installed                                                                                                                                                                        
✔ Ingress gateways installed                                                                                                                                                              
✔ Installation complete                                                                                                 
Made this installation the default for injection and validation.
```

查看安装的istio部分资源列表：

```shell
$ kubectl get deploy,svc,hpa,cm,secret -n istio-system
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           63m
deployment.apps/istiod                 1/1     1            1           68m

NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
service/istio-ingressgateway   LoadBalancer   20.1.124.145   <pending>     15021:30730/TCP,80:32703/TCP,443:32294/TCP   63m
service/istiod                 ClusterIP      20.1.237.53    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        68m

NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          63m
horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 <unknown>/80%   1         5         1          68m

NAME                                            DATA   AGE
configmap/istio                                 2      68m
configmap/istio-ca-root-cert                    1      5m49s
configmap/istio-gateway-status-leader           0      5m49s
configmap/istio-leader                          0      5m49s
configmap/istio-namespace-controller-election   0      5m49s
configmap/istio-sidecar-injector                2      68m
configmap/kube-root-ca.crt                      1      68m

NAME                     TYPE               DATA   AGE
secret/istio-ca-secret   istio.io/ca-root   5      5m51s

# 查看istio安装的自定义资源
$ kubectl -n istio-system get crd
NAME                                                  CREATED AT
authorizationpolicies.security.istio.io               2024-03-08T00:43:31Z
bgpconfigurations.crd.projectcalico.org               2024-03-03T05:22:57Z
bgpfilters.crd.projectcalico.org                      2024-03-03T05:22:57Z
bgppeers.crd.projectcalico.org                        2024-03-03T05:22:57Z
...

# 查看istio安装的role和rolebinding
$ kubectl -n istio-system get role,rolebinding
NAME                                                      CREATED AT
role.rbac.authorization.k8s.io/istio-ingressgateway-sds   2024-03-08T00:48:32Z
role.rbac.authorization.k8s.io/istiod                     2024-03-08T00:43:32Z

NAME                                                             ROLE                            AGE
rolebinding.rbac.authorization.k8s.io/istio-ingressgateway-sds   Role/istio-ingressgateway-sds   68m
rolebinding.rbac.authorization.k8s.io/istiod                     Role/istiod                     73m
```

检查安装结果：

```shell
$ ./istioctl verify-install manifest.yaml                           
1 Istio control planes detected, checking --revision "default" only
✔ HorizontalPodAutoscaler: istio-ingressgateway.istio-system checked successfully
✔ Deployment: istio-ingressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-ingressgateway.istio-system checked successfully
...
✔ Service: istiod.istio-system checked successfully
✔ ServiceAccount: istiod.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istio-validator-istio-system.istio-system checked successfully
Checked 15 custom resource definitions
Checked 2 Istio Deployments
✔ Istio is installed and verified successfully

# 查看istio安装的API对象，istio使用它们来完成各项任务
$ kubectl api-resources |grep istio
wasmplugins                                    extensions.istio.io/v1alpha1           true         WasmPlugin
istiooperators                    iop,io       install.istio.io/v1alpha1              true         IstioOperator
destinationrules                  dr           networking.istio.io/v1beta1            true         DestinationRule
envoyfilters                                   networking.istio.io/v1alpha3           true         EnvoyFilter
gateways                          gw           networking.istio.io/v1beta1            true         Gateway
proxyconfigs                                   networking.istio.io/v1beta1            true         ProxyConfig
serviceentries                    se           networking.istio.io/v1beta1            true         ServiceEntry
sidecars                                       networking.istio.io/v1beta1            true         Sidecar
virtualservices                   vs           networking.istio.io/v1beta1            true         VirtualService
workloadentries                   we           networking.istio.io/v1beta1            true         WorkloadEntry
workloadgroups                    wg           networking.istio.io/v1beta1            true         WorkloadGroup
authorizationpolicies                          security.istio.io/v1                   true         AuthorizationPolicy
peerauthentications               pa           security.istio.io/v1beta1              true         PeerAuthentication
requestauthentications            ra           security.istio.io/v1                   true         RequestAuthentication
telemetries                       telemetry    telemetry.istio.io/v1alpha1            true         Telemetry
```

##### 8.4.3.2 关于IstioOperator

IstioOperator（官方有时简称为`iop`）是Istio规定的一个自定义资源（按K8s中的CRD格式），它允许用户配置和部署Istio服务网格的各个组件。
上节中我们安装的是istioctl推荐的默认配置，具体配置内容可通过`./istioctl profile dump`查看：

```shell
$ ./istioctl profile dump
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: false
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      name: istio-ingressgateway
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
...
```

其中的`spec.components`部分指定了需要安装（`enable: true`）的istio核心组件，
默认配置只安装了`base`,`ingressGateways`,`pilot`。
通过以下命令可以查看已安装的IstioOperator配置:

```shell
$ kubectl -n istio-system get IstioOperator installed-state -o yaml
```

每个组件的作用如下：

- base：istio核心组件之一，安装控制平面时必需；
- cni：istio后来添加的一个组件，用于代替原先注入到应用Pod的`istio-init`容器，以实现应用容器的流量劫持功能；
    - 具体来说，在部署应用Pod时，istio不再注入`istio-init`容器，而是使用CNI插件来为应用Pod配置网络。
    - 属于一项较为**先进**的功能，本文暂不使用；
- ingressGateways：网格流量的入站网关，用于管控网格的所有入站流量。比如TLS设置和路由策略等；
    - 常用组件，可用于替代K8s的ingress API；
- egressGateways：网格流量的出站网关，与ingress网关对称。用于管控网格内的所有出站流量；
    - 非必需组件，适用于以下场景：
        1. 管理员需要严格管控网格内某些应用的出站流量，这是一种安全需求；
        2. 集群内的节点没有公网IP，通过安装egress网关并为其配置公网IP，可以使网格内的流量以受控方式访问外部服务；
- istiodRemote：在受外部控制面管理的从集群中安装istio时需要的组件，
  可以参考[使用外部控制平面安装Istio][使用外部控制平面安装 Istio]；

- pilot：istio核心组件之一，负责网格内的服务发现和配置管理；
    - 服务发现：跟踪K8s集群中的Service资源变化，并实时下发给各个Envoy代理，避免各代理转发流量到失效主机；
    - 配置管理：包括网格内的路由规则和流量规则等，它们以K8s CRD的形式存储在集群中。
    - pilot划分为两个组件：
        - pilot-discovery：位于istiod内部，负责发现集群内的配置变化和Service端点变化，并通过gRPC stream连接实时下发给Envoy代理；
        - pilot-agent：位于Envoy代理内部，负责从pilot-discovery获取最新的证书、策略配置以及服务端点，并将其注入到Envoy代理的配置文件中；

其他关于IstioOperator配置的操作命令:

```shell
# 查看内置的配置列表，查看每个配置的介绍：https://istio.io/latest/zh/docs/setup/additional-setup/config-profiles/
# 实战中根据需求选择哪个档位。
$ istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    minimal
    openshift
    preview
    remote
# 查看demo档位的配置内容
$ istioctl profile dump demo

# 查看配置的某个部分
$ istioctl profile dump --config-path components.pilot demo
# 查看不同档位配置间的差异，例如default和demo
$ istioctl profile diff default demo

# 卸载istio
$ istioctl uninstall --purge && kubectl delete ns istio-system
```

最后，IstioOperator自有一套配置规范，请查阅[IstioOperator Options](https://istio.io/latest/zh/docs/reference/config/istio.operator.v1alpha1)。

#### 8.4.4 Envoy介绍

Envoy是一个由C++开发的、开源的、面向大型现代服务架构设计的**高性能**L7代理和通信总线。它于2016年8月开源，2017年9月捐献给CNCF开始孵化，
后在2018年11月成为CNCF继 Kubernetes 与 Prometheus 之后第三个毕业的项目。Envoy的定位是做一个高性能的通用代理，可以与任何语言构建的服务通信，
它能够提供服务发现、负载均衡、TLS终止、L3(TCP/UDP等) & HTTP/2 & gRPC代理、熔断、追踪、健康检查、限流等丰富的网络通信转发功能。

> Envoy还支持代理MongoDB、Redis和Postgres协议。

- [了解Envoy的完整介绍](https://cloudnative.to/envoy/intro/what_is_envoy.html)

虽然Istio采用其作为默认的数据平面代理，但Envoy并不与Istio绑定。Envoy通常以sidecar模式部署在应用Pod中，但也支持作为前端/边缘代理角色部署，就像网关一样。
参考[此页面](https://jimmysong.io/kubernetes-handbook/usecases/envoy-front-proxy.html)了解Envoy如何作为前端代理进行工作。

**显著优势**

- 性能：Envoy是C++开发的高性能L7代理；
- 丰富特性：通过配置各类过滤器执行丰富的流量和路由策略；
- 支持多种协议：支持HTTP/1.1、HTTP/2、gRPC、MongoDB、Redis、MySQL、Postgres、DNS等协议；
- 动态配置：通过xDS API协议动态接收来自控制面下发的配置，实现免重启更新配置。同时也支持静态配置，在作为前端/边缘代理时会用到。
    - xDS（`* Discovery Service`的缩写），其包含L(istener)DS、C(luster)DS、R(oute)DS、E(ndpint)DS、S(ecret)DS等在内的多个发现服务。

**四大组件**  
Envoy内部有四个主要组件，下面根据工作顺序按序介绍：

- 监听器（Listener）：类似Nginx的监听端口，接收来自外部的TCP入站连接请求；
- 过滤器（Filter）：处理监听器接收到的入站请求所传输的流量；
    - 主要是通过预配置的各种过滤器进行工作，可以进行请求路由、速率限制等操作；多个过滤器之间通过链式配置和工作；
    - 丰富的过滤器类别是Envoy的核心功能优势。
- 路由（Router）：指定如何将入站连接请求路由到Envoy代理内部配置的Cluster，这是一个流量出站步骤；
- 集群（Cluster）：这里的**集群**不同于传统意义上的集群，而是指流量转发的目的地址（类似Nginx的upstream，即转发的后端地址），同时可以配置负载均衡等策略。

**常用术语**

- 主机（Host）：一个具有网络通信能力的端点，例如服务器、移动智能设备等，而这里的 host 通常代表上游服务
- 集群（Cluster）：集群是Envoy连接到的一组逻辑上相似的端点；在v2中，RDS通过路由指向集群，CDS提供集群配置，而Envoy通过EDS发现集群成员，即端点（Endpoint）；
- 下游（Downstream）：下游主机连接到 Envoy，发送请求并接收响应，它们是 Envoy 的客户端；
- 上游（Upstream）：上游主机接收来自 Envoy 的连接和请求并返回响应，它们是 Envoy 代理的后端服务器，可能是容器、虚拟机、服务器；
- 端点（Endpoint）：端点即上游主机，是一个或多个集群的成员，可通过 EDS 发现；
- 侦听器（Listener）：侦听器是能够由下游客户端连接的命名网络位置，例如端口或 unix 域套接字等；
- 子集（Subset）：子集是具有相同标签的端点的逻辑分组，例如金丝雀发布时通常会根据版本标签配置两个不同版本的服务子集，
  然后为两个子集分配不同的流量权重（可能是1:9）；
- 位置（Locality）：上游端点运行的区域拓扑，包括地域、区域和子区域等；
- 管理服务器（Management Server）：实现 v3 API 的服务器，它支持复制和分片，并且能够在不同的物理机器上实现针对不同 xDS API 的
  API 服务。Istio中指控制面；
- 地域（Region）：区域所属地理位置；
- 区域（Zone）：AWS 中的可用区（AZ）或 GCP 中的区域等；
- 子区域：Envoy 实例或端点运行的区域内的位置，用于支持区域内的多个负载均衡目标；
- Mesh和Envoy Mesh：指代服务网格，由多个基于envoy代理的sidecar构成。

> 如何快速理解上下游概念？
> - 一般只对代理这个角色讨论上下游概念；
> - 进入代理的请求即是下游请求，而代理再转发的则是上游请求。例如Nginx反代，Client发出的是下游请求，Nginx转发至Server的是上游请求，
    所以Nginx配置中使用`upstream`关键字来定义Server端点；
> - 注意：理解此概念后才能在后续的演示练习中游刃有余的分析Envoy日志。

#### 8.4.5 使用演示

##### 8.4.5.1 sidecar注入的两种方式

Istio通过为对象添加标签（Label）的方式来实现sidecar注入，具体分为自动和手动两种：

- 自动注入：通过为K8s集群中的命名空间（Namespace）对象添加Istio识别的特定标签来实现自动注入；
    - 具体来说，当K8s集群中的命名空间对象被添加了`istio-injection=enabled`标签时，任何新的 Pod 都将在创建时自动添加sidecar。
    - 这种注入对工作负载是完全无感知的，它不会修改资源模板；
    - 若不希望Pod被注入sidecar，则可以为Pod对象添加`sidecar.istio.io/inject="false"`标签（此标签优先级**高于**命名空间标签）；
- 手动注入：通过手动执行命令实现sidecar注入
    - 具体来说，执行`istioctl kube-inject -f samples/sleep/sleep.yaml`命令即可将sidecar注入到指定的Pod
    - 以上命令使用istio安装到集群的配置进行注入，也可以使用本地配置，具体方法参考[手动注入][手动注入]；
    - 由于对资源模板的入侵性，**手动注入仅在特定情况下使用**。

本文仅介绍更常用的自动注入方式，操作步骤如下：

```shell
# 为命名空间设置特定标签，以 default 为例
$ kubectl label namespace default istio-injection=enabled --overwrite
namespace/default labeled
$ kubectl get ns default --show-labels                               
NAME         STATUS   AGE    LABELS
default      Active   2m2s   istio-injection=enabled,kubernetes.io/metadata.name=default

# 删除标签命令：kubectl label namespace default istio-injection-
```

添加标签后，新部署的Pod都会自动注入sidecar，但不会修改资源清单。

##### 8.4.5.2 部署Pod进行验证

这里仍然使用第一节中只定义了一个常规容器的 [deployment.yaml](k8s_actions_guide/version1/base_manifest/deployment.yaml)
模板进行验证，注意还需要部署相关的几个资源：

- [configmap.yaml](k8s_actions_guide/version1/base_manifest/configmap.yaml)
- [secret.yaml](k8s_actions_guide/version1/base_manifest/secret.yaml)
- [service.yaml](k8s_actions_guide/version1/expose_manifest/service.yaml)
    - Istio的mTLS仅在定义了service的工作负载上生效，所以必须部署service。

操作步骤如下：

```shell
# 1. 部署configmap、secret以及service，步骤略。

# 2. 部署deployment
$ kubectl apply -f deployment.yaml

# 3. 可以看到 READY 处的2/2表示Pod内部有2个容器正在运行
$ kubectl get po -l app=go-multiroute
NAME                            READY   STATUS    RESTARTS   AGE
go-multiroute-b6fcdf544-7xfn7   2/2     Running   0          2m40s
go-multiroute-b6fcdf544-dqncx   2/2     Running   0          5m27s

# 4. 查看Pod事件，可以看到Pod启动过程中创建了注入的istio-init和istio-proxy容器，加上原本的 go-multiroute 一共3个容器
# -- 但其中的istio-init是初始化容器
$ kubectl describe po go-multiroute-b6fcdf544-7xfn7 |grep Events: -A 15
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m16s  default-scheduler  Successfully assigned default/go-multiroute-b6fcdf544-7xfn7 to k8s-node1
  Normal  Pulled     2d3h   kubelet            Container image "docker.io/istio/proxyv2:1.20.3" already present on machine
  Normal  Created    2d3h   kubelet            Created container istio-init
  Normal  Started    2d3h   kubelet            Started container istio-init
  Normal  Pulled     2d3h   kubelet            Container image "docker.io/leigg/go_multiroute:v1" already present on machine
  Normal  Created    2d3h   kubelet            Created container go-multiroute
  Normal  Started    2d3h   kubelet            Started container go-multiroute
  Normal  Pulled     2d3h   kubelet            Container image "docker.io/istio/proxyv2:1.20.3" already present on machine
  Normal  Created    2d3h   kubelet            Created container istio-proxy
  Normal  Started    2d3h   kubelet            Started container istio-proxy
  
# 5. 查看istio正在代理的所有Pod，包括每个Pod的xDS同步状态，可添加 -n 指定 namespace
$ ./istioctl proxy-status                        
NAME                                      CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                     VERSION
go-multiroute-b6fcdf544-7xfn7.default     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-78478fdc7-qmzbb     1.20.3
go-multiroute-b6fcdf544-dqncx.default     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-78478fdc7-qmzbb     1.20.3
```

##### 8.4.5.3 Istio特性之mTLS

Istio提供诸多特性，这里将介绍如何使用其中的**mTLS**（双向TLS）特性。
启用mTLS可以实现sidecar之间的**双向身份认证、传输流量加密**，可以避免中间人攻击，提高传输安全性。
部署后，Istio的控制面可以自动完成将要过期的sidecar证书轮换。
要启用mTLS，必须启动两个不同的服务进行互相通信，前面已经部署了一个`go-multiroute`服务，
它通过configmap提供了两个HTTP接口`/route1`和`/route2`，这里将其作为被调用端，
然后我们再启动一个调用端服务[istio_client_test.yaml](k8s_actions_guide/version1/istio_manifest/istio_client_test.yaml)，
构建和上传镜像的步骤略，包含代码实现在内的相关清单在 [istio_manifest/](k8s_actions_guide/version1/istio_manifest) 目录下。

操作步骤如下：

```shell
# 为了避免网络策略的干扰，先删除之前部署的默认拒绝 networkpolicy
kubectl delete networkpolicy block-internal-all-egress
kubectl delete networkpolicy deny-all-ingress

# 部署调用端Pod
kubectl apply -f istio_client_test_pod.yaml

# 查看启动日志（启动时发起调用）
$ kubectl logs istio-client-test-$POD_ID                                       
2024/03/08 08:07:22 status:200 text: [v1] Hello, You are at /route1, Got: route1's content

# 然后可在istio-client-test中的Pod上进行抓包测试。
# - 使用tcpdump抓取默认网卡上来源是3000端口的tcp数据流，可以观察到包含"Hello"字眼的http响应明文
# - （开始抓包后，需要同步在相同的Pod上的另一个容器中执行curl访问go-multiroute服务：kubectl exec -it istio-client-test-$POD_ID -c istio-client-test -- curl go-multiroute:3000/route1）
# - （执行curl访问后，这边应立即可以观察到控制台输出相关tcp数据）
$ kubectl exec -it istio-client-test-$POD_ID -c tcpdump -- tcpdump -nX tcp src port 3000
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
09:43:58.668073 IP 20.2.36.112.3000 > 20.2.36.117.60022: Flags [P.], seq 3671286507:3671287547, ack 2394647540, win 291, options [nop,nop,TS val 2452817 ecr 2452816], length 1040
...
	0x0040:  204f 4b0d 0a64 6174 653a 2053 756e 2c20  .OK..date:.Sun,.
	0x0050:  3130 204d 6172 2032 3032 3420 3039 3a34  10.Mar.2024.09:4
	0x0060:  333a 3538 2047 4d54 0d0a 636f 6e74 656e  3:58.GMT..conten
...
	0x00e0:  643a 2073 6964 6563 6172 7e32 302e 322e  d:.sidecar~20.2.
	0x00f0:  3336 2e31 3132 7e67 6f2d 6d75 6c74 6972  36.112~go-multir
	0x0100:  6f75 7465 2d62 3666 6364 6635 3434 2d62  oute-b6fcdf544-b
	0x0110:  7a7a 7237 2e64 6566 6175 6c74 7e64 6566  zzr7.default~def
	0x0120:  6175 6c74 2e73 7663 2e63 6c75 7374 6572  ault.svc.cluster
	0x0130:  2e6c 6f63 616c 0d0a 782d 656e 766f 792d  .local..x-envoy-
	0x0140:  7065 6572 2d6d 6574 6164 6174 613a 2043  peer-metadata:.C
	0x0150:  6945 4b44 6b46 5155 4639 4454 3035 5551  iEKDkFQUF9DT05UQ
...
	0x03e0:  5555 5344 786f 4e5a 3238 7462 5856 7364  UUSDxoNZ28tbXVsd
	0x03f0:  476c 7962 3356 305a 513d 3d0d 0a73 6572  Glyb3V0ZQ==..ser
	0x0400:  7665 723a 2069 7374 696f 2d65 6e76 6f79  ver:.istio-envoy
	0x0410:  0d0a 0d0a 4865 6c6c 6f2c 2059 6f75 2061  ....Hello,.You.a
	0x0420:  7265 2061 7420 2f72 6f75 7465 312c 2047  re.at./route1,.G
	0x0430:  6f74 3a20 726f 7574 6531 2773 2063 6f6e  ot:.route1's.con
	0x0440:  7465 6e74                                tent
```

现在为 `go-multiroute`
服务配置双向认证策略（mTLS）：[peer_authn.yaml](k8s_actions_guide/version1/istio_manifest/peer_authn.yaml)，
它同样是以K8s资源清单的形式部署，该资源的`kind`为`PeerAuthentication`。操作步骤如下：

```shell
# 部署mTLS策略
$ kubectl apply -f peer_authn.yaml           
peerauthentication.security.istio.io/go-multiroute created

# 可使用缩写形式：pa
$ kubectl get peerauthentications
NAME            MODE     AGE
go-multiroute   STRICT   4m42s
```

部署后，访问 `go-multiroute` 服务就必须遵循策略中配置的强制（STRICT）双向TLS通信模式。
这可以通过上面使用过的tcpdump抓包方式来验证，抓取结果将是各种乱码，无法观察到人眼可读的明文。

> 启用mTLS特性需要注意两点：
>- 对已存在的服务启用mTLS，由于服务可能正在被调用，
   > 需要先设置策略为`PERMISSIVE`模式（同时接收明文和HTTPS流量）进行过渡，
   > 等待相关服务更新（建议等待1min）后再切换到`STRICT`模式。
>- 若要关闭mTLS，**直接删除策略是不起作用的**，必须将策略改为`DISABLE`模式然后应用，这样才会生效。
   > 同理，为了避免影响正在运行的服务，需要先设置策略为`PERMISSIVE`模式进行过渡，一段时间后再切换到`DISABLE`模式。
>- Istio支持为纯TCP或基于TCP的HTTP、gRPC等协议提供mTLS支持；**Istio不会代理UDP协议**（但Envoy本身支持），
   > 即应用容器的UDP数据报文不会经过sidecar容器的网络栈，所以也不需要在Istio的流量策略中配置应用的UDP端口；

其他建议和Tips：

- 可以设置根命名空间`istio-system`的默认mTLS模式为`STRICT`
  以影响整个网格范围，使用模板 [peer_authn_default.yaml](k8s_actions_guide/version1/istio_manifest/peer_authn_default.yaml)。
- 将模板中的`namespace`字段改为非根命名空间以影响特定命名空间；
- 策略优先级：工作负载级别 > 命名空间级别 > 根命名空间级别;

> 对等认证即mTLS是Istio提供的两个认证特性之一，另一个认证特性是请求认证（`RequestAuthentication`），它使用JWT技术实现针对HTTP请求的认证。
> 由于篇幅所限和使用频率不高，本文不会详述，
> 请参考[请求认证](https://istio.io/latest/zh/docs/concepts/security/#request-authentication)。

**清理**

```shell
kubectl delete -f peer_authn.yaml
```

##### 8.4.5.4 Istio特性之授权

Istio通过预先安装到Kubernetes集群中的`AuthorizationPolicy`自定义API资源来提供细粒度的授权功能，其具有如下优点和特性：

- 实现**拒绝或允许**指定的不同`通信源`之间的流量访问；
- `通信源`可以是一组Pod/IP块/服务账号，甚至是一整个命名空间；
- 可以限制具体执行的HTTP方法，如GET、POST等；
- 可以限制具体访问的HTTP路由，如`/get_order`；
- 大部分维度支持**排除匹配**，比如`to.operations.notPaths: /get_order`表示匹配除了`/get_order`以外的HTTP路由；
- 可以设置**工作负载、命名空间或网格维度**的默认授权策略，例如默认拒绝或允许范围内的所有HTTP通信；
- 可以为策略设置附加条件，例如若要拒绝通信源A到B之间的访问，还要求HTTP请求的Header中必须携带`version: v1`键值对；
- 支持HTTP 1.1/2以及HTTPS

> 注意：由于K8s内置的NetworkPolicy API与Istio的AuthorizationPolicy API存在功能上的重叠，为了降低架构复杂度，
> 建议一旦启用AuthorizationPolicy API，就尽量停止使用NetworkPolicy API（但可以使用NetworkPolicy来控制工作负载的UDP通信）。

一个`AuthorizationPolicy`资源清单由以下三部分组成：

- selector（选择器）：匹配一组要执行策略的目标；
- action（动作）：要执行的动作，可以`DENY`或`ALLOW`，另外还支持`CUSTOM`和`AUDIT`动作；
    - 包含`CUSTOM`动作的策略是将匹配后的访问信息**先**
      交给外部程序决断，若结果是允许，再由其他策略判断。并不常用，请参考 [CUSTOM授权][CUSTOM授权]
      和 [部署外部授权][部署外部授权]。
- rules（规则集合）：什么条件下控制谁到谁的访问，其中每个元素由三部分组成：
    - `from`字段指定请求来源，支持指定命名空间/认证身份/服务账号/IP块；
    - `to`字段指定请求方法、路由和端口；
    - `when`字段指定条件（触发时机），参考[Istio授权策略之`when`字段][Istio授权策略之`when`字段]；

**授权引擎的执行过程**

1. 若范围内没有创建任何策略，则允许；
2. 按照动作`CUSTOM`->`DENY`->`ALLOW`的顺序依次匹配策略，这个过程中**只要成功匹配一个拒绝策略则立即执行**；
3. 若没有创建任何`ALLOW`策略，则允许；
4. 最后，在定义了`ALLOW`策略的情况下却没有匹配到任何一个允许策略则拒绝。

**在普通 TCP 协议上使用 Istio 授权**  
Istio授权策略支持纯TCP和基于TCP的其他协议，若在针对纯TCP工作负载的授权策略中配置了仅HTTP支持的字段如`methods`等，
则Istio将忽略这些字段，但不影响其他字段的匹配行为。注意，所有授权策略都将下发给Envoy执行，所以这个执行过程是高效的。
仅HTTP支持的字段如下：

- `source.request_principals`
- `operation`部分中的 `hosts`、`methods` 和 `paths` 字段
- `when.key`中的`request.headers`、`request.auth.claims/principal/audiences/presenter`

为了更接近最佳实践和实际环境，演示将完成以下目标（提供模板）：

- 创建default空间下的默认拒绝策略；
    - [authz-allow-nothing.yaml](k8s_actions_guide/version1/istio_manifest/authz-allow-nothing.yaml)；
    - 注意，这不会影响UDP通信；
- 创建一个允许策略，允许①满足条件位于`other_ns`命名空间【或】服务帐户为`cluster.local/ns/default/sa/default`的工作负载
  **通过** ②`GET`和`POST` 方法访问③default命名空间下与`app: go-multiroute`标签匹配的工作负载提供的④以`/route`
  为前缀的路由，附加条件⑤是HTTP Header中包含键值对`version: v1`；
    - [authz-allow-to-go-multiroute.yaml](k8s_actions_guide/version1/istio_manifest/authz-allow-to-go-multiroute.yaml)

操作步骤如下：

```shell
# 首先部署默认拒绝策略
$ kubectl apply -f authz-allow-nothing.yaml 
authorizationpolicy.security.istio.io/allow-nothing created

# 此时 istio-client-test 测试访问 go-multiroute 会得到明确的拒绝访问提示！
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v1" go-multiroute:3000/route1
RBAC: access denied

# 部署第二个策略
$ kubectl apply -f authz-allow-to-go-multiroute.yaml                                    
authorizationpolicy.security.istio.io/allow-to-go-multiroute created

# 查看default空间中的授权策略列表
$ kubectl get authorizationpolicies                                                                                       
NAME                     AGE
allow-nothing            2m
allow-to-go-multiroute   2m

# 可以访问，因为满足 rules 中所要求的各种条件
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v1" go-multiroute:3000/route1
[v1] Hello, You are at /route1, Got: route1's content
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v1" go-multiroute:3000/route2
[v1] Hello, You are at /route2, Got: route2's content
$ kubectl exec -it istio-client-test-$POD_ID -- curl -X POST -H "version: v1" go-multiroute:3000/route2
[v1] Hello, You are at /route2, Got: route2's content

# --------- 观察下面被拒绝的情况 -----------
# - 拒绝访问（header不符合）
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v2" go-multiroute:3000/route1
RBAC: access denied
# - 拒绝访问（method不符合）
$ kubectl exec -it istio-client-test-$POD_ID -- curl -X PATCH -H "version: v1" go-multiroute:3000/route1
RBAC: access denied
# - 拒绝访问（路由不符合）
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v1" go-multiroute:3000
RBAC: access denied

# 修改istio-client-test使用的ServiceAccount为`sa-test`（创建SA的步骤略）
$ kubectl patch deployment istio-client-test -p '{"spec": {"template": {"spec": {"serviceAccountName": "sa-test"}}}}'
deployment.apps/istio-client-test patched
# ~等待deployment内Pod更新完毕
# - 拒绝访问（ServiceAccount不符合）
$ kubectl exec -it istio-client-test-$POD_ID -- curl -H "version: v1" go-multiroute:3000/route1
RBAC: access denied
# （为了方便下一节的演示，请回滚这次patch）
```

最后，本节中的示例并未列出可用的全部字段，如有兴趣请查看[Istio授权策略规范][Istio授权策略规范]。其他可供参考的文档：

- [为 Ingress 网关配置授权](https://istio.io/latest/zh/docs/ops/configuration/security/security-policy-examples/)
- [为 TCP 工作负载配置授权](https://istio.io/latest/zh/docs/tasks/security/authorization/authz-tcp/)

**清理**

```shell
kubectl delete -f authz-allow-nothing.yaml
kubectl delete -f authz-allow-to-go-multiroute.yaml
```

##### 8.4.5.5 Istio特性之流量管理

Istio的[流量管理][Istio流量管理]可以实现以下功能：

- 智能路由：对到达K8s Service的HTTP流量根据权重/HTTP-Header/sourceLabels等维度进行**分流**，基于此可轻松实现灰度发布、A/B
  测试、金丝雀发布等部署策略；
- 服务发现、健康检查：Envoy代理会主动发现需要关心的后端服务的端点列表，并通过健康检查移除失效的端点；
- 负载均衡：Envoy代理会根据预设的负载均衡算法（如随机、轮询、加权轮询和最少连接等）将请求均匀地分发到后端服务；
    - 在选择具体的负载均衡算法之前，Envoy支持三种类型的负载均衡器（不同类型支持不同算法）：
        - simple：基于常用负载均衡算法的简单负载均衡
        - consistentHashLB：基于一致性哈希算法的负载均衡
        - localityLbSetting：基于地域的本地性负载均衡
- 服务弹性：支持为后端配置限速、熔断、超时和重试等弹性能力；
- 故障注入：支持对后端服务进行故障注入，模拟服务故障，以便进行服务弹性测试。也可用来实现手动熔断的目的；
- 流量镜像：也称为影子流量，用于将实时流量的副本发送到另一组后端，一般用于线上流量分析、统计和程序测试等用途；
- Ingress和Egress：控制 Istio 服务网格的入站流量和入站流量，前者可以替代K8s内置的Ingress。

以上这些特性是通过Istio预安装的 `VirtualService` 和 `DestinationRule` 两个K8s API资源实现的，它们分别定义了路由规则和目标规则。
通俗来说，`VirtualService`定义了如何进行路由（How），`DestinationRule`定义了路由到哪儿（Where），
一定程度上是**前者引用后者**的关系。

> 注意：VirtualService和DestinationRule资源并不是强绑定的关系，它们可以各自定义并独立工作，
> 仅在定义subset（子集）时前者才会依赖后者。但只有将二者结合使用才能充分应用Istio丰富的流量管理特性。

为了尽可能模拟实际环境，本节演示内容将增加一个名为 `go-multiroute-v2` 的服务版本模拟金丝雀发布，
它与之前部署的版本区别在于模板中定义了不同的环境变量`VERSION`，相关资源如下：

- [deployment-v2.yaml](k8s_actions_guide/version1/base_manifest/deployment-v2.yaml)

新版本的工作负载的部署步骤略。本次演示将实现以下功能：

- 智能路由：对发送到Service：go-multiroute的流量进行按比例分流，即Deployment `go-multiroute` 和 `go-multiroute-v2`
  分别接收80%和20%的流量；同时将HTTP Header中携带 `test-version: v2` 的请求全部路由到`go-multiroute-v2`；
- 负载均衡：分流之后，对发送到每个Deployment控制下的Pod实例的流量进行负载均衡，使用随机算法；
- 服务弹性：对发送到Service：go-multiroute的流量限制最多5个请求的并发数，并配置熔断、超时机制；

新增下面的YAML清单：

- [route-destinationrule.yaml](k8s_actions_guide/version1/istio_manifest/route-destinationrule.yaml)
- [route-virtualservice.yaml](k8s_actions_guide/version1/istio_manifest/route-virtualservice.yaml)

操作步骤如下：

```shell
# 部署流量策略
$ kubectl apply -f route-destinationrule.yaml -f route-virtualservice.yaml
destinationrule.networking.istio.io/go-multiroute created
virtualservice.networking.istio.io/go-multiroute created

# 现在进入client pod验证各项配置是否生效
$ kubectl exec -it istio-client-test-$POD_ID -- sh

# 8比2的权重分流：访问10次，观察返回结果（每个结果的前4个字符表示响应的deployment版本）
/ $ for i in $(seq 1 10); do curl -s go-multiroute:3000/route1 && sleep 1 && echo; done
[v2] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content
[v1] Hello, You are at /route1, Got: route1's content

# 测试特定header的分流
/ $ for i in $(seq 1 5); do curl -s -H "test-version: v2" go-multiroute:3000/route1 && sleep 1 && echo; done
[v2] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content
[v2] Hello, You are at /route1, Got: route1's content

# 测试分流后的负载均衡：轮询（可观察到v1版本的ip分布是均衡的）
/ $ for i in $(seq 1 10); do curl -s go-multiroute:3000/get_ip && sleep 1 && echo; done
[v2] Hello, Your ip is 20.2.36.111
[v2] Hello, Your ip is 20.2.36.112
[v1] Hello, Your ip is 20.2.36.113
[v1] Hello, Your ip is 20.2.36.114
[v1] Hello, Your ip is 20.2.36.113
[v1] Hello, Your ip is 20.2.36.114
[v1] Hello, Your ip is 20.2.36.113
[v2] Hello, Your ip is 20.2.36.111
[v1] Hello, Your ip is 20.2.36.114
[v1] Hello, Your ip is 20.2.36.113

# 测试超时
# - 由于接口 /test_timeout 会等待3s再返回，超过了策略中的2s，所以会超时
/ $ for i in $(seq 1 3); do curl -s go-multiroute:3000/test_timeout && echo " $(date)"; done
upstream request timeout Thu Mar 14 05:47:29 UTC 2024
upstream request timeout Thu Mar 14 05:47:30 UTC 2024
upstream request timeout Thu Mar 14 05:47:31 UTC 2024

# 测试熔断
# - 由于Service：go-multiroute目前的端点较多（2个Deployment共4个Pod对应4个端点），为了方便观察测试结果，先删除deployment-v1（操作略）。
# - 现在只剩下deployment-v2，其中Pod副本数为2，将其改为1，操作略。而且负载均衡策略为轮询，根据如下熔断策略计算：
#    outlierDetection: # 定义熔断策略
#      consecutive5xxErrors: 3 # 指定连续多少个 5xx 错误会导致端点被剔除，默认5，0表示禁用（但是时间窗口未知）
#      interval: 1s # 熔断检测间隔，默认10s，要求>=1ms
#      baseEjectionTime: 10s # 初始的端点剔除时间，支持单位 ms s m h，默认30s，要求>=1ms
#      maxEjectionPercent: 100
# 即最少并发3(=1x3)个请求后，将触发熔断，10s后恢复（命令尾部的&表示并发）
# - 观察3个请求响应状态码均为500，因此会立即触发熔断，下一步进行验证
/ $ seq 3 | xargs -I {} sh -c 'curl -s -H "test-version:v2" -H "need_500:true" http://go-multiroute:3000/test_circuit_breaker &' && echo $(date)
Fri Mar 15 06:17:41 UTC 2024
[v2] test_circuit_breaker returned 500
[v2] test_circuit_breaker returned 500
[v2] test_circuit_breaker returned 500

# 立即执行如下命令验证熔断策略已触发
# - 下面的命令是以1次/1s的频率顺序执行12个HTTP请求
# - 观察前几个响应内容是来自Envoy的no healthy upstream，说明熔断策略已触发，并在约10s后恢复（之后又触发）
/ $ seq 12 | xargs -I {} sh -c 'curl -s -H "test-version:v2" http://go-multiroute:3000/test_circuit_breaker && echo " $(date)" && sleep 1'
no healthy upstream Fri Mar 15 06:17:44 UTC 2024
no healthy upstream Fri Mar 15 06:17:45 UTC 2024
no healthy upstream Fri Mar 15 06:17:46 UTC 2024
no healthy upstream Fri Mar 15 06:17:47 UTC 2024
no healthy upstream Fri Mar 15 06:17:48 UTC 2024
no healthy upstream Fri Mar 15 06:17:49 UTC 2024
no healthy upstream Fri Mar 15 06:17:50 UTC 2024
[v2] test_circuit_breaker returned 500
 Fri Mar 15 06:17:51 UTC 2024   <--- 约10s后恢复！
[v2] test_circuit_breaker returned 500
 Fri Mar 15 06:17:52 UTC 2024
[v2] test_circuit_breaker returned 500
 Fri Mar 15 06:17:53 UTC 2024
no healthy upstream Fri Mar 15 06:17:54 UTC 2024
no healthy upstream Fri Mar 15 06:17:55 UTC 2024

# 测试最大并发数①
# - 为了方便测试，这里暂时注释熔断策略配置，以及将deployment-v2的副本数改为1（操作略）
# - 进入默认容器内进行并发访问（配置最大并发数5，测试并发数8，结果中5个响应成功，3个超时）
/ $ seq 8 | xargs -I {} sh -c 'curl -s http://go-multiroute:3000/test_limiter &'
[v1] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
upstream request timeoutupstream request timeoutupstream request timeout

# 测试最大并发数②
# - 修改配置中的最大并发数为3，操作略
# - 可观察到测试并发数6，结果中3个响应成功，3个超时）
# - 之所以超时是因为该接口默认sleep 1s，对于非并发执行的请求，其响应耗时必将超过1s，则触发 VirtualService 中的timeout: 1200ms
/ $ seq 6 | xargs -I {} sh -c 'curl -s -H "test-version:v2" http://go-multiroute:3000/test_limiter &'
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
[v2] Hello, You are at /test_limiter, Got: test_limiter's content
upstream request timeoutupstream request timeoutupstream request timeout
```

**完整的金丝雀发布**

通过`VirtualService`API可以实现将流量路由到不同版本的服务（Pod组），这是第一步；分流以后，我们需要观察一段时间内v2版本服务的请求处理是否正常，这是第二步；
若不正常，则需要回滚，将流量全部路由回v1版本。若基本正常，则可以进行第三步，即修改`VirtualService`清单中的路由规则，
将所有流量仅路由到v2版本即可；最后，继续观察一段时间，确保v2版本服务稳定，则可以删除v1版本服务了。

**清理**

```shell
$ kubectl delete -f route-destinationrule.yaml -f route-virtualservice.yaml
destinationrule.networking.istio.io "go-multiroute" deleted
virtualservice.networking.istio.io "go-multiroute" deleted
```

##### 8.4.5.6 流量管理之Ingress网关

与K8s内置的Ingress类似，如果你按照前面的[安装步骤](#8431-安装)Istio在安装时也部署了一个Ingress
网关（本质是Envoy）来管理集群的入站流量，它可以用来替换K8s内置的Ingress，
好处是它具有更细粒度的控制功能。

> 注意：K8s Ingress也可与Istio网格协同工作。

如果没有安装Ingress网关，使用`istioctl install`命令安装即可。使用如下命令检查是否安装：

```shell
$ kubectl get deploy,svc,hpa -n istio-system |grep gateway -B 1
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           3d7h
--
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
service/istio-ingressgateway   LoadBalancer   20.1.185.158   <pending>     15021:31132/TCP,80:31227/TCP,443:30215/TCP   3d7h
--
NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          3d7
```

可见Ingress网关是以Deployment形式部署，并且还创建了一个HPA以供Pod副本弹性伸缩。最后，Istio为Ingress网关创建了一个LoadBalancer类型的Service，将其对外暴露。
但由于笔者的集群是自建的，所以无法为LoadBalancer类型的Service分配一个外部的公网IP，但我们可以使用它的 `CLUSTER-IP`
字段即`20.1.185.158`来做集群内的访问测试。

与K8s Ingress不同的是，Istio Ingress网关的编程方式是一个Gateway 资源绑定一个VirtualService 资源的组合。当然也非常简单，
你可以通过下面的演示来了解具体细节。

本节演示要实现的目标只有一个：将`Service: go-multiroute`的3000端口通过Ingress对外暴露到80和443端口，
使用证书对应的域名是：`*.foobar.com`，然后在任一集群节点上进行访问测试。
新增的资源清单文件如下：

- [ingress-gwy.yaml](k8s_actions_guide/version1/istio_manifest/ingress-gwy.yaml)
- [ingress-virtualservice.yaml](k8s_actions_guide/version1/istio_manifest/ingress-virtualservice.yaml)

具体操作步骤如下：

```shell
# 1. 因为使用了HTTPS协议，所以先为服务生成一个自签名证书

# - 1.1 生成根私钥和根证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
  -subj '/O=example Inc./CN=example.com' \
  -keyout ca.key -out ca.crt
openssl x509 -noout -text -in ca.crt # check crt

# - 1.2 使用根证书和根私钥为泛域名 *.foobar.com 生成私钥和证书（foobar.key | foobar.crt）
openssl genrsa -out foobar.key 2048
openssl req -key foobar.key -new -out foobar.csr -subj '/C=US/ST=California/CN=*.foobar.com/O=My Company'
openssl x509 -req -in foobar.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out foobar.crt -days 365

# 2. 创建网关使用的secret（只能在istio-system空间下创建才能被找到）
kubectl create secret tls cert-foobar --cert=foobar.crt --key=foobar.key -n istio-system

# - 使用istioctl查看网关Pod关联的secret（最后一个参数是<POD>.[ns]的格式）
$ ./istioctl pc secret istio-ingressgateway-d4db74f5b-l44h4.istio-system
RESOURCE NAME                TYPE           STATUS     VALID CERT     SERIAL NUMBER                        NOT AFTER                NOT BEFORE
default                      Cert Chain     ACTIVE     true           653e6e499c6dd8ac7309498fa7ee7dd5     2024-03-17T12:28:45Z     2024-03-16T12:26:45Z
kubernetes://cert-foobar     Cert Chain     ACTIVE     true           aaec8a8aee3fc3f9                     2024-04-15T15:49:40Z     2024-03-16T15:49:40Z
ROOTCA                       CA             ACTIVE     true           f915f3f5c5b3b5ffe27c8b3894d3d2bb     2034-03-11T04:58:57Z     2024-03-13T04:58:57Z


# 3. 创建Gateway和VirtualService
kubectl apply -f ingress-gwy.yaml -f ingress-virtualservice.yaml

# 使用istioctl查看网关路由
$ ./istioctl pc routes istio-ingressgateway-d4db74f5b-l44h4.istio-system
NAME          VHOST NAME                                       DOMAINS                                                   MATCH                  VIRTUAL SERVICE
http.8080     go-multiroute.default.svc.cluster.local:8080     go-multiroute.default.svc.cluster.local, *.foobar.com     /*                     ingress-go-multiroute.default
              backend                                          *                                                         /healthz/ready*        
              backend                                          *                                                         /stats/prometheus*

# 4.1 任何一个集群节点上进行HTTP访问
$ curl -H "Host: go-multiroute.default.svc.cluster.local" 20.1.185.158/route1  
[v2] Hello, You are at /route1, Got: route1's content
$ curl -H "Host: x.foobar.com" 20.1.185.158/route1 
[v2] Hello, You are at /route1, Got: route1's content
$ curl 20.1.185.158/route1 -w %{http_code}% # 不带匹配主机名的Host头部会被当做404处理
404

# 4.2 现在进行HTTPS访问
$ echo '20.1.185.158 go-multiroute.default.svc.cluster.local' >> /etc/hosts
$ echo '20.1.185.158 x.foobar.com' >> /etc/hosts
# -k 禁用客户端对服务器证书的检查（因为根证书没有对go-multiroute...这个地址生成过证书）
$ curl -k -H "Host: go-multiroute.default.svc.cluster.local" https://go-multiroute.default.svc.cluster.local/route1
[v2] Hello, You are at /route1, Got: route1's content
$ curl --cacert ./ca.crt https://x.foobar.com/route1   
[v2] Hello, You are at /route1, Got: route1's content
```

演示完成。下面我们简单看一下网关的一些运行时信息：

```shell
# listener是当前网关Pod（Envoy）运行时监听的端口以及对应服务。这里前两行就是我们刚才的配置
$ ./istioctl pc listener istio-ingressgateway-d4db74f5b-l44h4.istio-system
ADDRESSES PORT  MATCH                                                     DESTINATION
0.0.0.0   8080  ALL                                                       Route: http.8080
0.0.0.0   8443  SNI: *.foobar.com,go-multiroute.default.svc.cluster.local Route: https.8443.https.ingress-go-multiroute.default
0.0.0.0   15021 ALL                                                       Inline Route: /healthz/ready*
0.0.0.0   15090 ALL                                                       Inline Route: /stats/prometheus*

# routes是当前网关Pod（Envoy）运行时维护的路由规则，这里可以看到我们刚才配置的两条规则
# - 注意观察【VIRTUAL SERVICE】列，该列非404即说明网关内配置的目标成功匹配到了定义的虚拟路由！
$ ./istioctl pc routes istio-ingressgateway-d4db74f5b-l44h4.istio-system
NAME                                               VHOST NAME                                       DOMAINS                                                   MATCH                  VIRTUAL SERVICE
http.8080                                          go-multiroute.default.svc.cluster.local:8080     go-multiroute.default.svc.cluster.local, *.foobar.com     /*                     ingress-go-multiroute.default
https.8443.https.ingress-go-multiroute.default     go-multiroute.default.svc.cluster.local:8443     go-multiroute.default.svc.cluster.local, *.foobar.com     /*                     ingress-go-multiroute.default
                                                   backend                                          *                                                         /healthz/ready*        
                                                   backend                                          *                                                         /stats/prometheus*     

# clusters是当前网关Pod（Envoy）运行时维护的后端（envoy中称为cluster）列表
$ ./istioctl pc clusters istio-ingressgateway-d4db74f5b-l44h4.istio-system
SERVICE FQDN                                            PORT      SUBSET     DIRECTION     TYPE           DESTINATION RULE
BlackHoleCluster                                        -         -          -             STATIC         
agent                                                   -         -          -             STATIC         
go-multiroute.default.svc.cluster.local                 3000      -          outbound      EDS            
istio-ingressgateway.istio-system.svc.cluster.local     80        -          outbound      EDS            
istio-ingressgateway.istio-system.svc.cluster.local     443       -          outbound      EDS            
istio-ingressgateway.istio-system.svc.cluster.local     15021     -          outbound      EDS            
istiod.istio-system.svc.cluster.local                   443       -          outbound      EDS            
istiod.istio-system.svc.cluster.local                   15010     -          outbound      EDS            
istiod.istio-system.svc.cluster.local                   15012     -          outbound      EDS            
istiod.istio-system.svc.cluster.local                   15014     -          outbound      EDS            
kube-dns.kube-system.svc.cluster.local                  53        -          outbound      EDS            
kube-dns.kube-system.svc.cluster.local                  9153      -          outbound      EDS            
kubernetes.default.svc.cluster.local                    443       -          outbound      EDS            
prometheus_stats                                        -         -          -             STATIC         
sds-grpc                                                -         -          -             STATIC         
xds-grpc                                                -         -          -             STATIC         
zipkin                                                  -         -          -             STRICT_DNS  

# endpoint是网关Pod（Envoy）运行时维护的cluster背后的所有端点健康状态（比如一个service后面可能有几个pod，就都会展示在这里）
$ ./istioctl pc endpoint istio-ingressgateway-d4db74f5b-l44h4.istio-system
ENDPOINT                                                STATUS      OUTLIER CHECK     CLUSTER
127.0.0.1:15000                                         HEALTHY     OK                prometheus_stats
127.0.0.1:15020                                         HEALTHY     OK                agent
192.168.31.2:6443                                       HEALTHY     OK                outbound|443||kubernetes.default.svc.cluster.local
20.2.36.83:53                                           HEALTHY     OK                outbound|53||kube-dns.kube-system.svc.cluster.local
20.2.36.83:9153                                         HEALTHY     OK                outbound|9153||kube-dns.kube-system.svc.cluster.local
20.2.36.86:8080                                         HEALTHY     OK                outbound|80||istio-ingressgateway.istio-system.svc.cluster.local
20.2.36.86:8443                                         HEALTHY     OK                outbound|443||istio-ingressgateway.istio-system.svc.cluster.local
20.2.36.86:15021                                        HEALTHY     OK                outbound|15021||istio-ingressgateway.istio-system.svc.cluster.local
20.2.36.89:3000                                         HEALTHY     OK                outbound|3000||go-multiroute.default.svc.cluster.local
20.2.36.90:53                                           HEALTHY     OK                outbound|53||kube-dns.kube-system.svc.cluster.local
20.2.36.90:9153                                         HEALTHY     OK                outbound|9153||kube-dns.kube-system.svc.cluster.local
20.2.36.91:15010                                        HEALTHY     OK                outbound|15010||istiod.istio-system.svc.cluster.local
20.2.36.91:15012                                        HEALTHY     OK                outbound|15012||istiod.istio-system.svc.cluster.local
20.2.36.91:15014                                        HEALTHY     OK                outbound|15014||istiod.istio-system.svc.cluster.local
20.2.36.91:15017                                        HEALTHY     OK                outbound|443||istiod.istio-system.svc.cluster.local
unix://./etc/istio/proxy/XDS                            HEALTHY     OK                xds-grpc
unix://./var/run/secrets/workload-spiffe-uds/socket     HEALTHY     OK                sds-grpc

# 它们都可以通过一些参数进行过滤
$ ./istioctl pc endpoint istio-ingressgateway-d4db74f5b-l44h4.istio-system --port 3000
ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
20.2.36.92:3000     HEALTHY     OK                outbound|3000||go-multiroute.default.svc.cluster.local
20.2.36.94:3000     HEALTHY     OK                outbound|3000||go-multiroute.default.svc.cluster.local
```

最后，有一个小的细节需要注意，删除包含证书的secret并不会影响网关继续正常响应对应证书域名的请求，
这是因为网关不会处理删除证书的操作，除非网关重启重新加载这些数据。所以要删除一个对外的服务端口，我们必须删除对应的Gateway配置。
此外，在部署验证过程中，观察网关Pod的日志可以确认网关是否成功加载了新配置。

**清理**

```shell
kubectl delete -f ingress-virtualservice.yaml -f ingress-gwy.yaml
```

**扩展主题**

- [配置双向TLS入站网关](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-mutual-tls-ingress-gateway)
- [当为多个 Gateway 配置了相同的 TLS 证书导致 404 异常](https://istio.io/latest/zh/docs/ops/common-problems/network-issues/#not-found-errors-occur-when-multiple-gateways-configured-with-same-TLS-certificate)
- [不发送 SNI 时配置 SNI 路由](https://istio.io/latest/zh/docs/ops/common-problems/network-issues/#configuring-SNI-routing-when-not-sending-SNI)

##### 8.4.5.7 流量管理之访问外部服务

Ingress网关控制的是外部流量如何进入网格，但网格如何访问外部服务呢？目前的默认做法是sidecar没有针对未知目标服务的流量代理做出任何限制，
比如应用容器可以直接访问类似`www.google.com`的外部域名或IP，这种不受控的对外访问会带来安全风险，
尤其在某些组织要求对所有对外流量进行加密时。

Istio提供了三种方式来解决这个问题：

1. 配置sidecar代理拒绝调用未知的目标服务
2. 配置ServiceEntry来声明外部服务，然后就可以通过VirtualService和DestinationRule来控制网格如何访问外部服务
3. 对于特定IP范围的目标服务，配置Istio使应用容器绕过sidecar代理

本节内容主要参考自[Istio：访问外部服务][Istio访问外部服务]，其中第一种和第三种方式操作起来都很简单，所以本节演示仅限于第二种方式，
读者可以结合本节演示和官文加深对ServiceEntry的理解。

本节演示将实现如果通过ServiceEntry+VirtualService的方式来管理网格如何访问外部的一个专用于HTTP测试的网站`httpbin.org`，
所使用的清单是 [external-access-control.yaml](k8s_actions_guide/version1/istio_manifest/external-access-control.yaml)，
演示步骤如下：

```shell
# 部署两种对象
$ kubectl apply -f external-access-control.yaml
serviceentry.networking.istio.io/httpbin created
virtualservice.networking.istio.io/httpbin created

# 验证一：根路由重定向到 /ip （-L跟随跳转）
$ kubectl exec -it istio-client-test-$POD_ID -- curl -L httpbin.org
{
  "origin": "119.x.198.51"
}
# 验证二：设置超时2s，要求延时3s返回（预期超时）
$ kubectl exec -it istio-client-test-$POD_ID -- curl httpbin.org/delay/3 -I
HTTP/1.1 504 Gateway Timeout
content-length: 24
content-type: text/plain
date: Mon, 18 Mar 2024 09:59:12 GMT
server: envoy

# 验证三：设置超时2s，要求延时1s返回（预期正常200）
$ kubectl exec -it istio-client-test-$POD_ID -- curl httpbin.org/delay/1 -I
HTTP/1.1 200 OK
date: Tue, 19 Mar 2024 13:29:55 GMT
content-type: application/json
content-length: 1473
server: envoy
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 1557
```

**理解原理**

创建Istio ServiceEntry对象可以帮助我们将一个外部的域名注册到Istio sidecar中，之后就可以在VirtualService中引用该域名并编写路由规则。
若没有注册外部服务就直接引入外部Host到路由策略中，创建对象不会报错，但是应用在访问时会得到503的状态码结果。

ServiceEntry对象支持为指定Host声明多个IP地址以及分别为每个IP指定端口，免去了DNS查询的步骤，方便管理仅通过IP公开的外部服务，
具体请参考其编写[规范](https://istio.io/latest/docs/reference/config/networking/service-entry/)。

##### 8.4.5.8 流量管理之Egress网关

Ingress网关和Egress网关共同实现了网格网络的东西向流量控制。但相比Ingress而言，Egress网关并不算常用，
因为大部分系统并不会要求对出口流量做控制，除非是一些特殊的、对网络出站安全有特别管控的系统。
也是由于sidecar的存在，Istio可以很轻松的实现对出站流量的管控。这里的管控包含以下功能：

- 针对指定域名的出站流量的流量管理（熔断/超时等）
- 针对指定域名的出站流量的加密（TLS）
    - 例如某些项目不允许服务器对外发起HTTP访问，所以当app发起HTTP请求时，请求到达Egress网关后，Egress网关将按策略对目标发起HTTPS请求
- 统一监控并记录网格服务的出站流量（在Egress网关处查询即可）

> K8s虽然没有提出Egress网关的概念，但提供了NetworkPolicy来控制Egress流量，其原理是通过CNI即网络插件来实现的（部分功能需要安装的CNI插件支持），
> 但所提供的功能丰富性无法与Istio相提并论。

此外，Istio还提及了Egress网关的另一个使用场景是：“集群中的应用节点没有公有 IP，所以在该节点上运行的网格 Service
无法访问互联网。通过定义Egress gateway，将公有IP分配给 Egress Gateway 节点，用它引导所有的出站流量，可以使应用节点以受控的方式访问外部服务。”

**安装Egress网关**

安装Egress网关，同时开启sidecar的访问日志

```shell
# 修改 iop 配置文件 istio-operator.yaml
# 此命令会更改已安装的名称为istio的configmap，并向网格中所有的Envoy实例注入新的配置。

#spec:
#  components:
#    egressGateways:
#    - enabled: true  
#      name: istio-egressgateway
#    ...
#  meshConfig:
#    ...
#    accessLogFile: /dev/stdout
#    accessLogEncoding: JSON

# 再次安装istio
$ ./istioctl install -f istio-operator.yaml

# 查看已安装的egress网关
$ kubectl get deploy,svc -n istio-system |grep egress -B 1
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-egressgateway    1/1     1            1           4m58s
--
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
service/istio-egressgateway    ClusterIP      20.1.68.34     <none>        80/TCP,443/TCP                               4m58s

# 查询sidecar的访问日志
$ kubectl exec -it istio-client-test-$POD_ID -- curl go-multiroute:3000/route1
[v2] Hello, You are at /route1, Got: route1's content

# 简单分析一下客户端sidecar的访问日志

# ①upstream是指sidecar代理的服务（也是转发目标），即sidecar1与sidecar2（远端节点）之间的连接
# client-app <---> sidecar1    ----------->    sidecar2 <---> server-app
# upstream_local_address -- sidecar1与sidecar2连接时的ip:port
# upstream_host -- sidecar2的ip:port
# upstream_cluster -- sidecar2的DNS信息

# ②downstream是指sidecar接收流量的客户端，即sidecar1与client-app（近端节点）之间的连接
# client-app <---> sidecar1    ----------->    sidecar2 <---> server-app
# downstream_remote_address：client-app的ip:port（与downstream_local_address建立连接）
# downstream_local_address：sidecar1的ip:port

# 其他还有response_code、route、duration、request_id、bytes_sent等有用信息。
$ kubectl logs -l app=istio-client-test -c istio-proxy --tail 1
{"upstream_transport_failure_reason":null,"protocol":"HTTP/1.1","user_agent":"curl/7.59.0","route_name":"default",
"response_code":200,"upstream_local_address":"20.2.36.96:39984","connection_termination_details":null,
"request_id":"5877d0cb-bd2b-4aa6-83c1-dd2e7dbf01be","response_flags":"-","upstream_host":"20.2.36.94:3000",
"bytes_sent":53,"x_forwarded_for":null,"upstream_cluster":"outbound|3000||go-multiroute.default.svc.cluster.local",
"authority":"go-multiroute:3000","start_time":"2024-03-17T16:42:07.449Z","method":"GET",
"response_code_details":"via_upstream","downstream_local_address":"20.1.207.122:3000","requested_server_name":null,
"duration":2,"upstream_service_time":"2","downstream_remote_address":"20.2.36.96:46218","path":"/route1","bytes_received":0}

# 此外，你还可以查看服务端istio-proxy的日志，并自行分析获得更清晰的访问记录认识。
```

sidecar默认的日志级别是Info，有时候可能需要为特定负载设置Debug级别日志，以便进行网络故障排查。可通过下面的方式设置：

```shell
# 方式一
kubectl exec {POD-NAME} -c istio-proxy -- curl -X POST http://127.0.0.1:15000/logging?level=debug
# 方式二
./istioctl pc log <POD-NAME>.[NAMESPACE] --level debug
```

本节演示包含两部分：

- 演示一：Egress网关透明转发HTTP请求（HTTP->HTTP）
    - 使用模板：[egressgwy-proxy-http2http.yaml][egressgwy-proxy-http2http.yaml]
- 演示二：Egress网关透明转发HTTPS请求（HTTPS->HTTPS）
    - 使用模板：[egressgwy-proxy-https2https.yaml][egressgwy-proxy-https2https.yaml]
- 演示三：Egress网关将HTTP请求转换为HTTPS请求（HTTP->HTTPS）
    - 使用模板：[egressgwy-proxy-http2https.yaml][egressgwy-proxy-http2https.yaml]
- ~~演示四：Egress网关将HTTPS请求转换为HTTP请求（HTTPS->HTTP）~~ 没有这种要求

[egressgwy-proxy-http2http.yaml]: k8s_actions_guide/version1/istio_manifest/egressgwy-proxy-http2http.yaml

[egressgwy-proxy-https2https.yaml]: k8s_actions_guide/version1/istio_manifest/egressgwy-proxy-https2https.yaml

[egressgwy-proxy-http2https.yaml]: k8s_actions_guide/version1/istio_manifest/egressgwy-proxy-http2https.yaml

**演示一：Egress网关透明转发HTTP请求（HTTP->HTTP）**

```shell
# 部署策略
$ kubectl apply -f egressgwy-proxy-http2http.yaml
serviceentry.networking.istio.io/istio-io created
gateway.networking.istio.io/egress-istio-io created
virtualservice.networking.istio.io/egressgateway-proxy-http-istio-io created

# 验证：在注入sidecar的客户端容器中访问注册的外部服务（预期：HTTP访问应该返回301）
$ kubectl exec -it istio-client-test-668bb6fc86-mmqf9 -- curl istio.io -I              
HTTP/1.1 301 Moved Permanently
content-type: text/plain; charset=utf-8
date: Wed, 20 Mar 2024 04:12:07 GMT
location: https://istio.io/
server: envoy
x-nf-request-id: 01HSD0YB5MSPEPC1Y5R85AMREM
x-envoy-upstream-service-time: 387
transfer-encoding: chunked

# 验证：查询egress网关日志，确认流量通过网关转发（根据 upstream_cluster 和 response_code 字段判断）
# - 在部署策略前，egress网关不会出现这条访问日志
# - 网关中的 upstream 是指自己所代理的服务，而egress代理的是外部服务；
$ kubectl logs -l istio=egressgateway -nistio-system --tail 2
{"requested_server_name":null,"authority":"istio.io","upstream_service_time":"376","path":"/","downstream_remote_address":"20.2.36.112:42936",
"x_forwarded_for":"20.2.36.112","upstream_local_address":"20.2.36.108:52900","downstream_local_address":"20.2.36.108:8080",
"duration":377,"bytes_sent":32,"upstream_transport_failure_reason":null,"response_code_details":"via_upstream",
"response_flags":"-","request_id":"252fce5c-0a92-99c2-93cd-4ce7ba0bd995","connection_termination_details":null,
"response_code":301,"upstream_cluster":"outbound|80||istio.io","start_time":"2024-03-18T16:52:08.157Z","bytes_received":0,
"upstream_host":"75.2.60.5:80","method":"GET","user_agent":"curl/7.59.0","route_name":null,"protocol":"HTTP/2"}
2024-03-18T16:35:39.362022Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012

# 清理
$ kubectl delete -f egressgwy-proxy-http2http.yaml
```

**演示二：Egress网关透明转发HTTPS请求（HTTPS->HTTPS）**

```shell
# 部署策略
$ kubectl apply -f egressgwy-proxy-https2https.yaml
serviceentry.networking.istio.io/istio-io-https created
gateway.networking.istio.io/egress-istio-io-https created
virtualservice.networking.istio.io/egressgateway-proxy-https-istio-io created

# 验证：查询egress网关日志，确认流量通过网关转发（根据 upstream_cluster 和 response_code 字段判断）
# - 在部署策略前，egress网关不会出现这条访问日志
$ kubectl logs -l istio=egressgateway -nistio-system --tail 1
{"upstream_local_address":"20.2.36.108:35544","duration":961,"requested_server_name":"istio.io","protocol":null,"path":null,
"method":null,"bytes_received":454,"authority":null,"connection_termination_details":null,"response_flags":"-","bytes_sent":3204,
"start_time":"2024-03-18T17:33:45.987Z","user_agent":null,"upstream_transport_failure_reason":null,"upstream_cluster":"outbound|443||istio.io",
"response_code_details":null,"upstream_service_time":null,"route_name":null,"upstream_host":"75.2.60.5:443","response_code":0,
"downstream_local_address":"20.2.36.108:8443","request_id":null,"x_forwarded_for":null,"downstream_remote_address":"20.2.36.112:52658"}

# 清理
$ kubectl delete -f egressgwy-proxy-https2https.yaml
```

**演示三：Egress网关将HTTP请求转换为HTTPS请求（HTTP->HTTPS）**

```shell
# 部署策略
$ kk apply -f egressgwy-proxy-http2https.yaml 
serviceentry.networking.istio.io/istio-io created
gateway.networking.istio.io/egress-istio-io-http2https created
destinationrule.networking.istio.io/egressgateway-for-istio-io created
virtualservice.networking.istio.io/egressgateway-proxy-istio-io-http2https created
destinationrule.networking.istio.io/originate-tls-for-istio-io created

# 验证：在客户端容器使用http协议访问host（预期会因为网关的转发而得到200响应）
# - 若没有网管的tls转发，则会得到301响应
$ kk exec -it istio-client-test-$POD_ID -- curl istio.io -I
HTTP/1.1 200 OK
...

# 清理
$ kubectl delete -f egressgwy-proxy-http2https.yaml
```

这三个演示可以覆盖大部分Egress网关的使用场景。对于演示三，我们需要知道网关是以`SIMPLE`方式对`istio.io`发起TLS连接，
这表示作为客户端的网关没有提供自己的证书给对方主机，因此存在第四个场景就是：网关与对方主机之间建立mTLS连接。
这个场景比较少见，它首先要求对方主机需要验证客户端证书，否则提供也没意义；如有兴趣，请参考官方提供的
[Egress网关发起的mTLS][Egress网关发起的mTLS演示] 演示。

[Egress网关发起的mTLS演示]: https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-gateway-tls-origination/#perform-mutual-TLS-origination-with-an-egress-gateway

**必要的安全措施**

请注意，Istio无法做到让集群中的所有出站流量都经过Egress网关转发，所以存在集群服务**绕过Egress网关直接访问外部服务**的可能，
这些服务的外站访问能够完全脱离Istio的控制和监控。

如果你不希望集群中有任何Pod的出站流量能够绕过Egress网关（脱离Istio的控制和监控），
建议你部署K8s的 NetworkPolicy 来保证**仅允许**来自Egress网关的流量可以通过集群边缘，
具体请参考[Istio Egress安全事项][Istio Egress安全]。

**仅允许访问已注册的服务**

如果你希望控制网格内的服务**仅能访问**已在网格或集群中注册的服务，那你可以通过修改 IstioOperator 配置文件来完成。执行命令：
`istioctl install -f istio-operator.yaml --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY`，默认的 `mode`
配置是`ALLOW_ANY`，即允许访问任何域名，而`REGISTRY_ONLY`表示**仅允许访问已注册的服务**。更改配置后，等待几秒钟生效，
然后你就会发现网格内的服务无法访问类似`api.alibaba.com`这样的外部地址了，因为它没有在网格内进行注册，
通过 ServiceEntry 对象完成网格内的服务注册就可以正常访问了。

最后，如果你在配置TLS时感到困惑，可以阅读官网文档
[理解 TLS 配置](https://istio.io/latest/zh/docs/ops/configuration/traffic-management/tls-configuration/)。

##### 8.4.5.9 其他示例

- 你可以将网格中的出站流量通过 ServiceEntry 引导至其他专用代理（而不是Egress网关），然后由专用代理负责控制和转发流量；
    - [使用外部HTTPS代理](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/http-proxy/)
- 上一节中的演示案例都是针对单一域名的引导配置，你也可以为一个泛域名创建 ServiceEntry 和 Gateway 等对象
    - [Wildcard 主机的 Egress](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/wildcard-egress-hosts/)
- 流量镜像：你可以配置将访问某个Host的流量按百分比复制到其他目标，这可以助你实现实施流量分析和安全审计；
    - [流量镜像](https://istio.io/latest/zh/docs/tasks/traffic-management/mirroring/)
- 速率限制：你可以为ingress网关和sidecar配置速率限制，以限制访问某个Host的流量速率；
    - [速率限制](https://istio.io/latest/zh/docs/tasks/policy-enforcement/rate-limit/)

此外，还有可观测性相关的更多示例，请参考[Istio 可观测性](https://istio.io/latest/zh/docs/tasks/observability/)。

#### 8.4.6 故障处理

**故障一：配置下发失败**

故障体现为应用容器没有安装配置的路由策略执行，通过`istioctl pc route ...`无法看到最新的路由策略。
此外观察istiod日志发现大量`error`级别的日志。

```shell
$ kubectl logs -l app=istiod -nistio-system --tail 5
2024-03-18T15:28:07.316091Z	error	security	Failed to authenticate client from 20.2.36.105:40058: 
Authenticator ClientCertAuthenticator: no verified chain is found; Authenticator KubeJWTAuthenticator: 
failed to validate the JWT from cluster "Kubernetes": the service account authentication returns an error: 
[invalid bearer token, service account token has expired]
...重复
```

具体原理笔者还未查明，但已经查到的问题是运行app容器的节点与K8s集群主节点时间不同步（相差好几个小时），
同步后无需操作集群，问题自动消失。

关于流量管理配置相关的故障，请参考官方文档[流量管理问题](https://istio.io/latest/zh/docs/ops/common-problems/network-issues/)。

#### 8.4.7 Istio服务的健康检查

Istio需要对APP容器的部分类型的**存活及就绪探针**规则进行重写，以使它们能够正常工作。这是因为：

- 对于HTTP类型的探针：kubelet负责向应用发送探测请求。但在启用Istio mTLS的情况下，由于kubelet没有Istio颁发的证书，因此健康检查会失败；
- 对于TCP类型的探针：kubelet仅检查Pod某端口是否开放。但在注入Istio sidecar的情况下，
  Pod已经被`istio-init`初始化容器所添加的iptables规则将从任意端口流入Pod的流量转发到sidecar处理，因为从外部探测Pod的任意TCP端口都会成功；
    - `iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15001`是一条示例iptables规则（实际会稍复杂点），
      它将所有进入Pod的TCP流量转发到sidecar的15001端口，即只要sidecar容器的15001端口正常监听，则外部对Pod的任意TCP端口探测都会成功。

其他类型的探针如Shell命令、gRPC类型的探针则不会被重写。当工作负载部署后，你可以查看其描述来了解重写后的规则。
当然，若你不希望为全局开启默认重写**或**想要为特定负载关闭规则重写行为，Istio也提供了对应的方法，具体操作较为简单，
请参考官方文档 [Istio 健康检查][Istio健康检查]。

#### 8.4.8 协议选择

首先，Istio 提供的sidecar代理支持原始TCP和基于TCP的HTTP系列协议以及gRPC协议。Istio 注入的sidecar不会代理UDP协议，
外部流量将直接到达APP容器而不会由sidecar转发。

> Istio 还实验性的对mongo、redis、mysql协议进行支持。

**显式的为Service指定协议类型**

当我们为应用定义K8s Service对象时，通常按照下面的模板定义：

```yaml
...
spec:
  type: ClusterIP
  selector:
    ...
  ports:
    - name: custom-name
      port: 80
      targetPort: 80
```

然而这种定义方式存在一个问题，那就是没有指定协议类型（K8s可能不需要，但Istio需要）。Istio仅支持对HTTP/HTTP2流量的识别，
对于其他TCP流量，统一当做不透明的原始TCP流量进入路由规则（VirtualService）匹配。此时，如果你提前为HTTPS流量定义了`tls`
类型的路由规则，那么很遗憾，这条规则不会生效，sidecar将按照默认方式转发流量，除了路由规则，一些高级流量策略（DestinationRule）也都将形同虚设。

因此，**最好显式的为Service指定协议类型**，Istio支持两种识别方式：

1. Service中的`ports.name`命名格式为`<protocol>[-<suffix>]`，例如`http`或`http-xxx`；
    - _不建议_。因为与sidecar不同，当Service作为Ingress网关的后端时，网关无法根据`ports.name`来识别协议类型，因此建议都使用第二种。
2. 在版本 v1.18+ 的Kubernetes，通过 `ports.appProtocol` 字段配置协议（优先于第一种）。

**备注**：本节内容是对官方文档 [Istio 协议选择][Istio协议选择] 更深度的解读与补充。

#### 8.4.9 流量路由详细过程

假设现在已经定义了一个 VirtualService（下简称VS） 和 DestinationRule（下简称DR） 对象，下面基于此假设进行阐述。

在 Istio 的流量路由中，我们将整个过程分为两个阶段：

- **匹配前端**：匹配待处理流量的协议类型
    - Istio 将创建的上述路由规则写入 sidecar 容器（或网关），可通过`istioctl pc route $SIDE_CAR_POD_ID`进行验证；
    - 当流量发送至 sidecar （或网关）时，后者根据 VS 中定义的流量协议进行匹配；
        - 若定义了`http`协议，则匹配HTTP/HTTP2/gRPC协议类型的流量（会检查[`Host`头部][HTTP-host-header]
          是否匹配，若是gRPC则检查[`:authority`头部][gRPC authority头部]）；
        - 若定义了`tls`协议，则匹配TLS协议类型（包含HTTPS和TCP over TLS）的流量（会在配置了 `tls.sniHosts`
          的情况下检查[`SNI`头部][what-is-sni]是否匹配，并且无法使用path和header相关的匹配条件）；
        - 若定义了`tcp`协议，则匹配原始TCP协议类型（包含以上所有协议）的流量（没有头部可供检查，但可以添加目标子网、端口或源负载标签的匹配条件）；
    - 若 sidecar （或网关）没有匹配到配置中的流量协议，则按以下流程处理流量:
        - 情况1：若是 ingress 网关，则返回404。若是 egress 网关，由于流量未到网关，则由sidecar自行处理（参照情况2）；
        - 情况2：若是 sidecar，则按本地路由表转发，目的地可能是网格内/外的服务，具体转发规则如下:
            - 若是网格内服务，则可能使用 mTLS 进行转发（前提是配置了相应的 mTLS 策略）；
            - 其他情况则应该是统一当做不透明的TCP流量处理。
- **转发后端**：决定以何种协议转发流量至后端
    - 当 sidecar （或网关）成功匹配到流量协议后，会根据 VS 中定义的`route.destination`进行转发，进入下一步；
    - 若 VS 中为转发后端还定义了`subset`（那该后端一定是集群内定义的Service），则查找 Host 对应的 DR 规则，根据 DR
      规则中定义的`subset`进行转发（若找不到对应的`subset`则返回404）；
        - 转发时遵循 DR 中定义的`spec.trafficPolicy`；
    - 若 VS 中没有定义`subset`，则也查找 Host 对应的 DR 规则，根据 DR 规则中定义的`spec.trafficPolicy`进行转发；
        - 若没有对应的 DR 规则，则按本地路由表转发；
    - 转发时需要决定使用何种协议类型：
        - 若后端是集群内定义的 Service，则使用 Service 定义中的`ports[?].appProtocol`或`ports.name`指示的协议；
        - 若后端是通过 Istio ServiceEntry 注册的外部服务，则使用 ServiceEntry 中定义的`ports[?].protocol`指示的协议；

**关于 Headless Service**

K8s中的Headless Service 是一种特殊的 Service，它没有 ClusterIP，而是通过DNS解析的方式，将 Service 的名称解析为一组 Pod 的
IP 地址。这种类型的 Service 作为 Istio 的转发后端时，Istio 要求它必须定义 `ports`字段，否则无法匹配。

## 参考

- [Kubernetes实战@美 Brendan Burns Eddie Villalba](https://book.douban.com/subject/35346815/)
- [Ambient Mesh 入门](https://istio.io/latest/zh/docs/ops/ambient/getting-started/)
- [Istio 服务网格：深入学习网络流量和架构](https://mp.weixin.qq.com/s/KXGvEN8obm3efsDdfHbgsA)
- [Enovy官方文档](https://cloudnative.to/envoy/)
- [云原生架构下的微服务之：envoy 原理基础](https://www.modb.pro/db/211373)
- [Istio 安全](https://istio.io/latest/zh/docs/concepts/security)
- [Istio 认证策略](https://istio.io/latest/zh/docs/tasks/security/authentication/authn-policy/)
- [Istio 授权](https://istio.io/latest/zh/docs/concepts/security/#authorization)
- [Envoy日志分析](https://www.zhaohuabing.com/istio-guide/docs/debug-istio/envoy-log/)

[MetricsAPI]: https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-api

[cadvisor]: https://learn.lianglianglee.com/专栏/由浅入深吃透%20Docker-完/08%20%20容器监控：容器监控原理及%20cAdvisor%20的安装与使用.md

[Introducing Ambient Mesh]: https://istio.io/v1.15/blog/2022/introducing-ambient-mesh/

[What’s a service mesh？And why do I need one?]: https://dzone.com/articles/whats-a-service-mesh-and-why-do-i-need-one

[Istio安装指南]: https://istio.io/latest/zh/docs/setup/install/

[使用外部控制平面安装 Istio]: https://istio.io/latest/zh/docs/setup/install/external-controlplane

[手动注入]: https://istio.io/latest/zh/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection

[Istio授权策略之`when`字段]: https://istio.io/latest/zh/docs/reference/config/security/conditions/

[CUSTOM授权]: https://istio.io/latest/zh/docs/reference/config/security/authorization-policy/#AuthorizationPolicy-Action

[部署外部授权]: https://istio.io/latest/zh/docs/tasks/security/authorization/authz-custom/

[Istio授权策略规范]: https://istio.io/latest/zh/docs/reference/config/security/authorization-policy/

[Istio流量管理]: https://istio.io/latest/zh/docs/concepts/traffic-management/

[Istio访问外部服务]: https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/#access-an-external-http-service

[Istio Egress安全]: https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-gateway/#additional-security-considerations

[Istio vs Linkerd]: https://imesh.ai/blog/istio-vs-linkerd-the-best-service-mesh-for-2023/

[Istio健康检查]: https://istio.io/latest/zh/docs/ops/configuration/mesh/app-health-check/

[Istio协议选择]: https://istio.io/latest/zh/docs/ops/configuration/traffic-management/protocol-selection/

[HTTP-host-header]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Host

[gRPC authority头部]: https://chromium.googlesource.com/external/github.com/grpc/grpc/+/HEAD/doc/PROTOCOL-HTTP2.md#protocol

[what-is-sni]: https://www.cloudflare.com/zh-cn/learning/ssl/what-is-sni/
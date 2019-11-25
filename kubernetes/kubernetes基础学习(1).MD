# kubernetes基础学习(1)

##一点题外话
什么是云原生？我想CNCF(Cloud Native Computing Foundation)的定义最具权威吧：
> 云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式API。
这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。
  
>云原生计算基金会（CNCF）致力于培育和维护一个厂商中立的开源生态系统，来推广云原生技术。我们通过将最前沿的模式民主化，让这些创新为大众所用。

下面进入正题
## kubernetes是什么？
先看官网定义：
>Kubernetes 是一个跨主机集群的 开源的容器调度平台，它可以自动化应用容器的部署、扩展和操作 , 提供以容器为中心的基础架构。
>
>使用 Kubernetes, 您可以快速高效地响应客户需求:
>
>+ 快速、可预测地部署您的应用程序
>
>+ 拥有即时扩展应用程序的能力
>
>+ 不影响现有业务的情况下，无缝地发布新功能
>
>+ 优化硬件资源，降低成本
>
>我们的目标是构建一个软件和工具的生态系统，以减轻您在公共云或私有云运行应用程序的负担。

## kubernetes组件
首先你要明白，k8s是由很多个二进制程序相互协作工作的，而这里的每一个二进制程序，都称为k8s的一个组件。它们有的来自k8s项目，有的来自k8s社区，
有的来自其他项目。
k8s组件整体如下图所示：
![kubernetes组件架构图](https://d33wubrfki0l68.cloudfront.net/817bfdd83a524fed7342e77a26df18c87266b8f4/3da7c/images/docs/components-of-kubernetes.png)
总体上，k8s组件分为两类：
+ master components：运行在master节点上的组件，包括kube-apiserver，etcd，kube-scheduler，kube-controller-manager；
+ node components：运行在每一台node节点上的组件，包括kubelet，kube-proxy，Container Runtime(Docker)
+ Addons：扩展组件，包括DNS，Web UI，Container Resource Monitoring，Cluster-level Logging


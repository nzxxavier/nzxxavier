# 1.K8S和Docker/Containerd的关系
K8S提供的是容器生命周期管理和容器编排的能力，他的核心还是调度能力，被调度的容器运行到对应的节点上还是需要节点上的容器运行时（Docker/containerd）来运行这个容器。
# 2.K8S核心组件
## 控制面
* kube-apiserver：访问Kubernetes系统的入口，以RESTful API接口方式提供给外部客户和内部组件调用，可供访问的资源包括原生的资源、插件安装的资源及用户定义的资源
* etcd：Kubernetes核心的键值对存储，存储元数据，kube-apiserver依赖etcd来提供服务
* kube-scheduler：根据资源、软件约束、亲和性、反亲和性等依据，调度pod到对应的node运行
* kube-controller-manager：Kubernetes自己实现的各种controller，包括PVC、Deployment、StatefulSet，等等的controller，用于提供集群基础功能
* cloud-controller-manager：能力与kube-controller-manager，提供一些云相关的控制能力，如Node Controller、Service Controller等
## 节点
* kubelet：节点的代理，负责沟通容器运行时把Pod对应的容器在节点上运行起来，容器生命周期也由kubelet管理
* kube-proxy：向iptables/ipvs管理Service相关的路由信息，也提供访问service时的路由、负载均衡能力，某些CNI插件可以替代kube-proxy，如Cilium或者Calico+eBFP模式
# 3.IPVS和iptables差别
都是基于linux内核的netfilter提供服务，差别在于iptables主要用于防火墙，IPVS用于高性能负载均衡，并使用更高效的数据结构（Hash表），所以在集群规模比较大的情况下，IPVS性能优于iptables
# 4.静态Pod
* 普通 Pod：由控制平面（API Server + 调度器）管理
* 静态 Pod：由节点上的 kubelet 直接管理，静态 Pod 的定义文件放在节点的特定目录中，kubelet 会监控这个目录并自动创建对应的 Pod
# 5.创建Pod的流程
  1. 客户端提交Pod的配置信息（可以是yaml文件定义的信息）到kube-apiserver
  2. Apiserver收到指令后，通知给controller-manager创建一个资源对象
  3. Controller-manager通过api-server将pod的配置信息存储到ETCD数据中心中
  4. Kube-scheduler检测到pod信息会开始调度预选，会先过滤掉不符合Pod资源配置要求的节点，然后开始调度调优，主要是挑选出更适合运行pod的节点，然后将pod的资源配置单发送到node节点上的kubelet组件上
  5. Kubelet根据scheduler发来的资源配置单运行pod，运行成功后，将pod的运行信息返回给scheduler，scheduler将返回的pod运行状况的信息存储到etcd数据中心
# 6.简述Kubernetes中Pod的重启策略
* Pod的重启策略包括Always、OnFailure和Never，默认值为Always。
* Always：当容器失效时，由kubelet自动重启该容器；
* OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器；
* Never：不论容器运行状态如何，kubelet都不会重启该容器。
* 同时Pod的重启策略与控制方式关联，当前可用于管理Pod的控制器包括ReplicationController、Job、DaemonSet及直接管理kubelet管理（静态Pod）。
# 7.不同控制器的重启策略限制
* RC和DaemonSet：必须设置为Always，需要保证该容器持续运行；
* Job：OnFailure或Never，确保容器执行完成后不再重启；
* kubelet：在Pod失效时重启，不论将RestartPolicy设置为何值，也不会对Pod进行健康检查。
# 8.Kubernetes有哪些负载控制器
``` text
Kubernetes Controller Manager
├── DeploymentController 
│   └── 管理 ReplicaSet
├── ReplicaSetController
│   └── 管理无状态 Pod
├── StatefulSetController
│   └── 直接管理有状态 Pod
├── DaemonSetController
└── JobController
```
# 9.Kubernetes中Pod的健康检查方式
* StartupProbe：检测容器内的应用程序是否已启动完成。在启动探针成功之前，其他探针会被禁用。
* ReadinessProbe：检测容器是否已准备好接收流量。如果探测失败，会将 Pod 从 Service 的端点列表中移除。
* LivenessProbe：检测容器是否正在运行。如果探测失败，kubelet 会重启容器。
# 10.探针可用的实现方式
* HttpGet：通过HTTP返回值判断，状态码大于等于200且小于400，则表明容器健康
* Exec：在容器内执行一个命令，若返回码为0，则表明容器健康
* TCPSocket：通过容器的IP地址和端口号执行TCP检查，若能建立TCP连接，则表明容器健康
# 11.Deployment升级的过程
1. 初始创建Deployment时，系统创建了一个ReplicaSet，并按用户的需求创建了对应数量的Pod副本
2. 当更新Deployment时，系统创建了一个新的ReplicaSet，并将其副本数量扩展到1，然后将旧ReplicaSet缩减为2
3. 之后，系统继续按照相同的更新策略对新旧两个ReplicaSet进行逐个调整
4. 最后，新的ReplicaSet运行了对应个新版本Pod副本，旧的ReplicaSet副本数量则缩减为0
# 12.Kubernetes deployment升级策略
* Recreate：设置spec.strategy.type=Recreate，更新Pod时，会先杀掉所有正在运行的Pod，然后创建新的Pod
* RollingUpdate：设置spec.strategy.type=RollingUpdate，滚动更新同时，可以通过设置spec.strategy.rollingUpdate下的两个参数（maxUnavailable和maxSurge）来控制滚动更新的过程
# 13.Kubernetes HPA原理
工作流：监控数据采集 → 指标聚合 → 期望副本数计算 → 执行扩缩容
数据流：
1. Pod 资源指标 (CPU/Memory) → Metrics Server → API Server → HPA Controller → Scale Subresource
2. Prometheus → Prometheus Adapter → Custom Metrics API → HPA Controller → Scale Subresource
# 14.简述Kubernetes Service类型
* ClusterIP：虚拟的服务IP地址，该地址用于Kubernetes集群内部的Pod访问，在Node上kube-proxy通过设置的iptables规则进行转发
* NodePort：使用宿主机的端口，使能够访问各Node的外部客户端通过Node的IP地址和端口号就能访问服务
* LoadBalancer：使用外接负载均衡器完成到服务的负载分发，需要在spec.status.loadBalancer字段指定外部负载均衡器的IP地址，通常用于公有云
# 15.Kubernetes Service轮询策略
* RoundRobin：默认为轮询模式，即轮询将请求转发到后端的各个Pod上，轮询实际上是由linux内核处理的，iptables和ipvs都支持配置轮询算法，iptables仅支持按权轮训，ipvs支持的算法较多
* SessionAffinity：基于客户端IP地址进行会话保持的模式，即第1次将某个客户端发起的请求转发到后端的某个Pod上，之后从相同的客户端发起的请求都将被转发到后端相同的Pod上
# 16.Kubernetes Headless Service
不提供负载均衡能力的Service，即不为Service设置ClusterIP（入口IP地址），仅通过Label Selector将后端的Pod列表返回给调用的客户端
# 17.数据持久化的方式
* EmptyDir：由K8S分配临时数据目录，生命周期与Pod一致，即Pod退出之后数据删除
* HostPath：将宿主机上已存在的目录或文件挂载到容器内部，类似于docker中的bind mount挂载方式
* PersistentVolume：如基于NFS服务的PV，也可以基于GFS的PV。它的作用是统一数据持久化目录，方便管理
# 18.Kubernetes所支持的存储供应模式
* 静态模式：集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置
* 动态模式：集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定
# 19.如何进行优雅的节点维护
节点排水，负载迁移
# 20.ETCD

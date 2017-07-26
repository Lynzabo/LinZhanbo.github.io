# Hello World总结

```reStructuredText
spec.selector是RC的Pod选择器，即监控和管理拥有这些标签（Label）的Pod实例，确保当前集群上始终有且仅有replicas个Pod实例在运行，这里我们设置replicas=1表示只能运行一个（名为redis-master的）Pod实例，当集群中运行的Pod数量小于replicas时，RC会根据spec.template段定义的Pod模板来生成一个新的Pod实例，labels属性指定了该Pod的标签，注意，这里的labels必须匹配RC的spec.selector，否则此RC就会陷入“只为他人做嫁衣”的悲惨世界中，永无翻身之时。
```



```
**创建Service时，K8s都会创建一个Pod间、内互通的一个虚拟VIP，该IP地址是由K8s系统自动分配的**，在其他Pod中无法预先知道某个Service的虚拟IP地址，因此需要一个机制来找到这个服务。为此，**K8s巧妙地使用了Linux环境变量，在每个Pod的容器里都增加了一组Service相关的环境变量，用来记录从Service名到虚拟IP地址的映射关系。**
```

# K8s衍生概念介绍

1. Node运行状态：包括Pending、Running、Terminated三种状态。

```
Node Controller是Kubernetes Master中的一个组件，用于管理Node对象。它的两个主要功能包括：集群范围内的Node信息同步，以及单个Node的生命周期管理。
```



```
当Kubelet的--register-node参数被设置为true（默认值即为true）时，Kubelet会向apiserver注册自己。这也是Kubernetes推荐的Node管理方式。
```



2. Pod是Kubernetes的最基本操作单元，包含一个或多个紧密相关的容器

```
一个Pod内所有容器在同一台主机上运行，不建议在Kubernetes的一个Pod内运行相同应用的多个实例。使用RC来管理Pod，RC中定义Pod副本数量。一组Pod构成了RC。
```



一个Pod中的应用容器共享同一组资源，如下所述：

1. PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID；
2. 网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围；
3. IPC命名空间：Pod中的多个容器能够使用SystemV IPC或者POSIX消息队列进行通信；
4. UTS命名空间：Pod中的多个容器共享一个主机名；
5. Volumes（共享存储卷）：Pod中的各个容器可以访问在Pod级别定义的Volumes。



Pod的生命周期是通过Replication Controller来管理的。Pod的生命周期过程包括：通过模板进行定义，然后分配到一个Node上运行，在Pod所含容器运行，RC结束后Pod也结束。在整个过程中，Pod处于一下4种状态之一：

1. Pending：Pod定义正确，提交到Master，但其所包含的容器镜像还未完成创建。通常Master对Pod进行调度需要一些时间，之后Node对镜像进行下载也需要一些时间；
2. Running：Pod已被分配到某个Node上，且其包含的所有容器镜像都已经创建完成，并成功运行起来；
3. Succeeded：Pod中所有容器都成功结束，并且不会被重启，这是Pod的一种最终状态；
4. Failed：Pod中所有容器都结束了，但至少一个容器是以失败状态结束的，这也是Pod的一种最终状态。



3. Label

```
Label以key/value键值对的形式附加到各种对象上，如Pod、Service、RC、Node等。Label定义了这些对象的可识别属性，用来对它们进行管理和选择。Label可以在创建时附加到对象上，也可以在对象创建后通过API进行管理。
在为对象定义好Label后，其他对象就可以使用Label Selector（选择器）来定义其作用的对象了。一般来说，我们会给一个Pod（或其他对象）定义多个Labels，以便于配置、部署等管理工作。例如：部署不同版本的应用到不同的环境中；或者监控和分析应用（日志记录、监控、告警）等。通过对多个Label的设置，我们就可以“多维度”地对Pod或其他对象进行精细的管理。
```



在某些对象需要对另一些对象进行选择时，可以将多个Label Selector进行组合，使用逗号","进行分隔即可。基于等式的Label Selector和基于集合的Label Selector可以任意组合。例如：

```shell
name=redis-slave,env!=production
name not in (php-frontend),env!=production
```



4. Replication Controller

```
Replication Controller是用于定义Pod副本的数量。在Master内，Controller Manager进程通过RC的定义来完成Pod的创建、监控、启停等操作。
根据Replication Controller的定义，Kubernetes能够确保在任意时刻都能运行用于指定的Pod“副本”（Replica）数量。如果有过多的Pod副本在运行，系统就会停掉一些Pod；如果运行的Pod副本数量太少，系统就会再启动一些Pod，总之，通过RC的定义，Kubernetes总是保证集群中运行着用户期望的副本数量。
可以说，通过对Replication Controller的使用，Kubernetes实现了应用集群的高可用性，并大大减少了系统管理员在传统IT环境中需要完成的许多手工运维工作（如主机监控脚本、应用监控脚本、故障恢复脚本等）。
```

RC扩容缩容例子：

```shell
 
[root@instance-xcul3 ~]# kubectl get rc
NAME           DESIRED   CURRENT   READY     AGE
frontend       3         3         3         19h
redis-master   1         1         1         20h
redis-slave    2         2         2         19h
[root@instance-xcul3 ~]# kubectl scale rc redis-slave --replicas=3
replicationcontroller "redis-slave" scaled
[root@instance-xcul3 ~]# kubectl get rc
NAME           DESIRED   CURRENT   READY     AGE
frontend       3         3         3         19h
redis-master   1         1         1         20h
redis-slave    3         3         2         19h
[root@instance-xcul3 ~]#
```

RC滚动更新：新版本+1，旧版本-1。

```shell
kubectl rolling-update frontend-v1 -f frontend-v2.json
```



5. Service

```
一个Service可以看作一组提供相同服务的Pod的对外访问接口。Service作用于哪些Pod是通过Label Selector来定义的。拥有一个唯一指定的名字。拥有一个虚拟IP（Cluster IP、Service IP、或VIP）和端口号。
```



```
在Pod正常启动后，系统将会根据Service的定义创建出与Pod对应的Endpoint（端点）对象，以建立起Service与后端Pod的对应关系。随着Pod的创建、销毁，Endpoint对象也将被更新。Endpoint对象主要由Pod的IP地址和容器需要监听的端口号组成。通过kubectl get endpoints命令可以查看，显示为Ip:Port的格式。
```

```shell
[root@instance-xcul3 helloworld]# kubectl get rc
NAME           DESIRED   CURRENT   READY     AGE
frontend       3         3         3         20h
redis-master   1         1         1         21h
redis-slave    3         3         3         21h
[root@instance-xcul3 helloworld]# kubectl get services
NAME           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
frontend       10.254.11.253    <nodes>       80:30001/TCP   20h
kubernetes     10.254.0.1       <none>        443/TCP        21h
redis-master   10.254.228.156   <none>        6379/TCP       21h
redis-slave    10.254.165.49    <none>        6379/TCP       20h
[root@instance-xcul3 helloworld]# kubectl get endpoints
NAME           ENDPOINTS                                         AGE
frontend       172.17.0.5:80,172.17.0.6:80,172.17.0.7:80         20h
kubernetes     192.168.0.12:6443                                 21h
redis-master   172.17.0.3:6379                                   21h
redis-slave    172.17.0.2:6379,172.17.0.4:6379,172.17.0.8:6379   20h
[root@instance-xcul3 helloworld]#
[root@instance-xcul3 helloworld]# ls
frontend-controller.yaml  redis-master-controller.yaml  redis-slave-controller.yaml
frontend-service.yaml     redis-master-service.yaml     redis-slave-service.yaml
[root@instance-xcul3 helloworld]# kubectl delete -f frontend-service.yaml
service "frontend" deleted
[root@instance-xcul3 helloworld]# kubectl get endpoints
NAME           ENDPOINTS                                         AGE
kubernetes     192.168.0.12:6443                                 21h
redis-master   172.17.0.3:6379                                   21h
redis-slave    172.17.0.2:6379,172.17.0.4:6379,172.17.0.8:6379   21h
[root@instance-xcul3 helloworld]#
```

```tex
	Pod的IP地址是Docker Daemon根据docker0网桥的IP地址段进行分配的，但Service的Cluster IP地址是Kubernetes系统中的虚拟IP地址，由系统动态分配。
	Service的Cluster IP地址相对于Pod的IP地址来说相对稳定，Service被创建时即被分配一个IP地址，在销毁该Service之前，这个IP地址都不会再变化了。而Pod在Kubernetes集群中生命周期较短，可能被ReplicationContrller销毁、再次创建，新创建的Pod将会分配一个新的IP地址。
	由于Service对象在Cluster IP Range池中分配到的IP只能在内部访问，所以其他Pod都可以无障碍地访问到它。但如果这个Service作为前端服务，准备为集群外的客户端提供服务，我们就需要给这个服务提供公共IP了。
	Kubernetes支持两种对外提供服务的Service的type定义：NodePort和LoadBalancer。
	NodePort:
     在定义Service时指定spec.type=NodePort，并指定spec.ports.nodePort的值，系统就会在Kubernetes集群中的每个Node上打开一个主机上的真实端口号。这样，能够访问Node的客户端都就能通过这个端口号访问到内部的Service了。
LoadBalancer:
    如果云服务商支持外接负载均衡器，则可以通过spec.type=LoadBalaner定义Service，同时需要指定负载均衡器的IP地址。使用这种类型需要指定Service的nodePort和clusterIP。
```



K8s的网络、Volume都时自己的。和Docker不同。

```tex
Volume是Pod中能够被多个容器访问的共享目录。Kubernetes的Volume概念与Docker的Volume比较类似，但不完全相同。Kubernetes中的Volume与Pod生命周期相同，但与容器的生命周期不相关。当容器终止或者重启时，Volume中的数据也不会丢失。另外，Kubernetes支持多种类型的Volume，并且一个Pod可以同时使用任意多个Volume。
EmptyDir
一个EmptyDir Volume是在Pod分配到Node时创建的。从它的名称就可以看出，它的初始内容为空。在同一个Pod中所有容器可以读和写EmptyDir中的相同文件。当Pod从Node上移除时，EmptyDir中的数据也会永久删除。

hostPath
在Pod上挂载宿主机上的文件或目录。

gcePersistentDisk
使用这种类型的Volume表示使用谷歌计算引擎上永久磁盘上的文件。

awsElasticBlockStore
nfs
iscsi
glusterfs
rbd
gitRepo
secret
一个secret volume用于为Pod提供加密的信息

persistentVolumeClaim
```



6. namespace

```tex
如果不特别指明Namespace，则用户创建的Pod、RC、Service都将被系统创建到名为“default”的Namespace中。用户可以根据需要创建新的Namespace，如在创建Pod时，可以指定Pod属于哪个Namespace。
```



# Kubernetes总体架构

Kubernetes将集群中的机器划分为一个Master节点和一群工作节点（Node）。

其中，

- etcd单独部署，是高可用的key/value存储系统，用于持久化存储集群中所有的资源对象，例如集群中的Node、Service、Pod、RC、Namespace等。
- 在Master上运行API Server（进程kube-apiserver）、Controller Manager（进程kube-controller-manager）和Scheduler（进程kube-scheduler）三个组件，这些进程实现了整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错等管理能力，并且都是全自动完成的。
- 在每个Node上运行Kubelet、Proxy（进程kube-proxy）和Docker Daemon三个组件，这些服务进程负责Pod的创建、启动、监控、重启、销毁以及实现软件模式的负载均衡器。
- 另外在集群内所有节点上都可以运行Kubectl命令行工具，它提供了Kubernetes的集群管理工具集。

## RC与相关Service创建流程

下面我们以RC与相关Service创建的完整流程为例，来说明K8s里各个服务（组件）的作用以及它们之间的交互关系。

创建RC过程：

1. 我们通过kubectl提交一个创建RC的请求（假设Pod副本数为1），该请求通过API Server被写入etcd中，
2. 此时Controller Manager通过API Server的监听资源变化的接口监听到这个RC事件，分析后，发现当前及群众还没有它所需要的Pod实例，于是根据RC里的Pod模板定义生成一个Pod对象，通过API Server写入etcd中，
3. 接下来，此事件被Scheduler发现，它立刻执行一个复杂的调度流程，为这个新Pod选定一个落户的Node，可称这个过程为绑定（Pod Binding），然后又通过API Server将这一结果写入到etcd中，
4. 随后，目标Node上运行的kubelet进程通过API Server检测到这个“新生的”Pod并且按照它的定义，启动该Pod并任劳任怨地负责它的下半生，直到Pod的生命走到尽头。

创建Service过程：

1. 随后，我们通过kubectl提交一个映射到该Pod的Service的创建请求，该请求通过API Server被写入etcd中，
2. 此时Controller Manager通过API Server的监听资源变化的接口监听到这个RC事件，Controller Manager会通过Label标签查询到相关联的Pod实例，然后生成Service的Endpoints信息（读取所有Pod的Docker daemon分配的IP）并通过API Server写入到etcd中。
3. 接下来，所有Node上运行的Proxy进程通过API Server查询并监听Service对象与其对应的Endpoints信息，建立一个软件方式的负载均衡器来实现Service访问到后端Pod的流量转发功能。

从上面的分析来看，K8s的各个组件的功能是很清晰的。

- API Server：提供了资源对象的唯一操作入口，其他所有组件都必须通过它提供的API来操作资源数据，通过对相关的资源数据“全量查询”+“变化监听”，这些组件可以很“实时”地完成相关的业务功能，比如某个新的Pod一旦被提交到API Server中，Controller Manager就会立刻发现并开始调度。
- Controller Manager：集群内部的管理控制中心，其主要目的是实现K8s集群的故障检测和恢复的自动化工作，比如根据RC的定义完成Pod的复制或移除，以确保Pod实例数符合RC副本的定义；根据Service与Pod的管理关系，完成服务的Endpoints对象的创建和更新；其他诸如Node的发现、管理和状态监控、死亡容器所占磁盘空间及本地缓存的镜像文件的清理等工作也是由Controller Manager完成的。
- Scheduler：集群中的调度器，负责Pod在集群节点中的调度分配。
- Kubelet：负责本Node节点上的Pod的创建、修改、监控、删除等全生命周期管理，同时kubelet定时“上报”本Node的状态信息到API Server里。
- Proxy：实现了Service的代理及软件模式的负载均衡器。

​        在K8s集群内部的客户端可以直接使用kubectl命令管理集群。

 Kubectl Proxy是API Server的一个反向代理，在K8s集群外部的客户端可以通过Kubectl Proxy来访问API Server。

 Kube-proxy进程为每个Service都建立一个”服务代理对象“，它包括一个用于监听此服务请求的SocketServer，SocketServer的端口是随机选择的一个本地空闲端口。此外，kube-proxy内部也创建了一个”负载均衡器组件“，用来实现SocketServer上收到的连接到后端多个Pod连接之间的负载均衡和会话保持能力。具体执行过程看后面“kube-proxy进程”节详细介绍。

 API Server内部有一套完备的安全机制，包括认证、授权及准入控制等相关模块。API Server在收到一个REST请求后，会首先执行认证、授权和准入控制的相关逻辑，过滤掉非法请求，然后将请求发送给API Server中的REST服务模块去执行资源的具体操作逻辑。

 在Node节点运行的Kubelet服务中内嵌了一个cAdvisor服务，cAdvisor是谷歌的另外一个开源项目，用于实时监控Docker上运行的容器的性能指标，在第4章会详细介绍它。

```tex

1. Master端进程

访问API Server提供Node、Service、Pod等Rest接口
K8s对API调用使用CA（Client Authentication）、Token和HTTP Base方式实现用户认证。
目前使用的Token方式，API Server启动时指定token_auth_file文件，curl请求API接口，指定header Authorization。

Authorization授权
启动API Server时，可指定哪些操作接口只允许特定用户操作。leengine现在没有。

Admission Controller准入机制
指定API Server启动会加载哪些插件，比如启动ResourceQuota插件，当在某个namespace创建ResourceQuota，指定限制创建资源数量如RC数量/等或者资源请求总量如CPU总量等，或者LimitRanger插件，当在某个namespace创建LimitRange，列举每个容器CPU最大/最小值，每个Pod最大最小值。
Secret私密凭据
使用Opaque的Secret
使用.dockercfg的Secret
使用Service Account的Secret
Secret的重要作用是保管私密数据，比如密码、OAuth Tokens、SSH Keys等信息。将这些私密信息放在Secret对象中比直接放在Pod或Docker Image中更安全，也更便于使用。
在使用Mount方式挂载Secret(上面volumes有一种类型是secret类型)时，Container中Secret的“data”域的各个域的Key值作为目录中的文件，Value值被Base64编码后存储在相应的文件中。在指定volume的挂载目录下。

Kubectl Proxy进程
考虑到安全策略，访问K8s的Master，必须基于TOKEN文件或客户端证书及HTTP Base认证，K8s提供了一个代理程序——Kubectl Proxy，它既能作为Kubernetes API Server的返回代理，也能作为普通客户端访问API Server的代理。

Controller Manager（kube-controller-manager）进程
Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）等的管理并执行自动化修复流程，确保集群处于预期的工作状态。比如在出现某个Node意外宕机时，Controller Manager会在集群的其它节点上自动补齐Pod副本。

Replication Controller
确保在任何时候集群中一个RC锁关联的Pod都保持一定数量的Pod副本处于正常运行状态。

Node Controller
负责发现、管理和监控集群中的各个Node节点，Kubelet在启动时通过API Server注册节点信息，并定时向API Server发送节点信息。

ResourceQuota Controller
资源配额管理确保了指定的对象在任何时候都不会超量占用系统资源。用户通过API Server请求创建或修改资源，API Server会调用Admission Controller的ResourceQuota插件，该插件灰度去前面写入etcd的配额统计结果，如果某向资源的配额已经被使用完，则此请求会被拒绝。

Namespace Controller
定时通过API Server读取这些Namespace信息。如果Namespace被API标识为优雅删除（设置删除期限，DeletionTimestamp属性被设置），则将该Namespace的状态设置成“Terminating”并保存到etcd中。同时Namespace Controller删除该Namespace下的ServiceAccount、RC、Pod、Secret、PersistentVolume、ListRange、ResourceQuota和Event等资源对象。

ServiceAccount Controller与Token Controller

Service Controller与Endpoint Controller
在创建Service时，如果指定了标签选择器（在spec.selector域中指定），那么系统会自动创建一个和该Service同名的Endpoint资源对象。该Endpoint资源对象包含一个地址（Address）和端口（Ports）集合。这些IP地址和端口号是通过标签选择器过滤出来的Pod的访问端点。K8s支持通过TCP和UDP去访问这些Pod的地址，默认使用TCP。

如何通过虚拟IP访问到后端Pod呢？
在K8s集群中的每个节点上都运行着一个叫作“kube-proxy”的进程，该进程会观察K8s Master节点添加和删除“Service”和“Endpoint”的行为。kube-proxy为每个Service在本地主机上开一个端口（随机选择）。任何访问该端口的连接都被代理到相应的一个后端Pod上。kube-proxy根据Round Robin算法及Service的Session粘连（SessionAffinity）决定哪个后端Pod被选中。最后，kube-proxy在本地的Iptables中安装相应的规则，这些规则使得Iptables将捕获的流量（通过Cluster IP和Port访问的Service的请求）重定向到前面提及的随机端口。通过该端口流量再被kube-proxy转到相应的后端Pod上。

​ 在创建了服务后，服务Endpoint模型会创建后端Pod的IP和端口列表（包含在Endpoints对象中），kube-proxy就是从这个Endpoints列表中选择服务后端的。集群内的节点通过虚拟IP和端口能够访问Service后端的Pod。
​ 默认情况下，K8s会为Service指定一个集群IP（或虚拟IP、cluster IP），但在某些情况下，用户希望能够自己指定该集群IP。为了给Service指定集群IP，用户只需要在定义Service时，在Service的spec.clusterIP域中设置所需要的IP地址即可。为Service指定的IP地址必须在集群的CIDR范围内。如果该IP地址是非法的，那么API Server会返回422 HTTP状态码，表明IP地址值非法。

K8s支持两种主要的模式来找到Service：一种是容器的Service环境变量，另一种是DNS。

K8s集群内访问任意服务

通过容器的Service环境变量模式：
​ 在创建一个Pod时，Kubelet在该Pod中的所有容器中为当前所有Service添加一系列环境变量。K8s既支持Docker links变量，也支持形如“{SERNAME}SERVICE_HOST”和“{SVCNAME}SERVICE_PORT”的变量。其中“{SVCNAME}”是大写的Service Name，同时Service Name包含的“-”符号会转化成“_”符号。

通过名称找到服务的方式是DNS：
​ DNS服务器通过K8s API监控与Service相关的活动。当监控到添加Service时，DNS服务器为每个Service创建一系列DNS记录。例如，在K8s集群的“my-ns”Namespace中有一个叫做“my-service”的Service，用“my-service”通过DNS应该能够访问到“my-ns” Namespace中的后端Pod。如果在其他Namespace中访问这个Service，则用“my-service.my-ns”来查找该Service。DNS返回的查询结果是集群IP（虚拟IP、cluster IP）。

K8s集群外访问任意服务
集群外部用户希望Service能够提供一个供集群外部用户访问的IP地址，甚至是公网IP地址，通过该IP来访问集群内的Service。
​ K8s提供了两种方式，一个是“NodePort”，另一个是“LoadBalancer”。
如果在定义Service时，设置spec.type的值为“NodePort”，则K8s Master节点将为Service的nodePort指派一个端口范围（默认为30000~32767），每个节点的kube-proxy指定一个端口，则可以在Service定义中通过指定spec.ports.nodePort域的值实现，你所指定的端口必须在前面提及的端口范围中。这个端口就不需要kube-proxy随机生成了，并且kube-proxy会在iptables配置关于这个端口的连接都映射到kube-proxy进程。


Scheduler（kube-scheduler）进程
Scheduler的作用是将待调度的Pod（API新创建的Pod、Controller Manager为补足副本而常见的Pod等）按照特定的调度算法和调度策略绑定（Binding）到集群中的某个合适的Node上，并将绑定信息写入etcd中。

Kubernetes Scheduler当前提供的默认调度流程分为如下两步：
  1.预先调度过程，即遍历所有目标Node，筛选出符合要求的候选节点。因此，K8s内置了多种预选策略（xxx Predicates）供用户选择。
  2.确定最优节点，在第一步的基础上，采用优选策略（xxx Priority）计算出每个候选节点的积分，积分最高者胜出。

2. 请求访问API Server的客户端
Kubectl进程
Java语言Client Libraries

3. Slave端进程
Kubelet进程
  节点管理
  Pod管理
  容器健康检查
  cAdvisor资源监控
  Heapster作为pod监控K8s集群，cAdvisor已集成到kubelet中，对主机上所有容器/主机监控，使用influxdb存储Heapster数据。使用Grafana可以展示Heapster。
Heapster作为Pod运行在K8s集群中，和运行在K8s集群中的其他应用相似。Heapster Pod通过Kubelet发现所有运行在集群中的节点，并查看来自这些节点的资源使用状况信息。Kubelet通过cAdvisor获取其所在节点及该节点上所有容器的数据，Heapster通过带着关联标签的Pod分组这些信息，这些数据被推到一个可配置的后端，用于存储和可视化展示。当前支持的后端包括InfluxDB。
​ cAdvisor是一个开源的分析容器资源使用率和性能特性的代理工具。自然支持Docker容器。在K8s项目中，cAdvisor被集成到Kubelet代码中。
​ cAdvisor自动查找所有在其所在节点上的容器，自动采集CPU、内存、文件系统和网络使用的统计信息。cAdvisor通过它所在节点机的Root容器，采集并分析该节点机的全面的使用情况。

Kube-proxy进程
Kube-proxy进程为每个Service都建立一个”服务代理对象“，它包括一个用于监听此服务请求的SocketServer，SocketServer的端口是随机选择的一个本地空闲端口。此外，kube-proxy内部也创建了一个”负载均衡器组件“，用来实现SocketServer上收到的连接到后端多个Pod连接之间的负载均衡和会话保持能力。下面详细讲解。
Kubernetes会基于Pod级别创建一个网络栈，同一个Pod内的不同容器将会共享一个网络命名空间，同一个Pod内的容器可以通过localhost来连接对方的端口。


Kubernetes的网络实现

容器到容器的通信
在同一个Pod内的容器（Pod内的容器是不会跨宿主机的）共享同一个网络命名空间，共享同一个Linux协议栈。可以用localhost地址访问彼此的端口。

Pod之间的通信
同一个Node内的Pod之间的通信：
每一个Pod都有一个真实的全局IP地址（K8s的CNM分配的每个Pod的），同一个Node内的不同Pod之间可以直接采用对方Pod的IP地址通信，而且不需要使用其他发现机制，例如DNS、Consul或者etcd。
同一个机器上Pod都连的docker0网桥，在同一个子网内，所以可以互通。

不同Node上的Pod之间的通信：
可以使用桥接/直接路由/flannel等跨主机容器通信方案。

Pod到Servcice之间的通信
1. Cluster IP+NodePort，节点IP+NodePort访问服务，kube-proxy发生了什么？
​ 访问Service的请求，不论是用ClusterIP+TargetPort的方式，还是用节点机IP+NodePort的方式，都被节点机的Iptables规则重定向到kube-proxy监听Service服务代理端口。
​ 此外，Service的Cluster IP与NodePort等概念是kube-proxy通过Iptables的NAT转换实现的，kube-proxy在运行过程中动态创建与Service相关的Iptables规则，这些规则实现了Cluster IP及NodePort的请求流量重定向到kube-proxy进程上对应服务的代理端口的功能。（首先kube-proxy会启一个SocketServer，随机监听一个端口，然后修改IPtables配置，所有请求Cluster IP+NodePort或节点IP+NodePort的请求都转发到这个端口的服务上。）由于Iptables机制针对的是本地的kube-proxy端口，所以如果Pod需要访问Service，则它所在的那个Node上必须运行kube-proxy，并且在每个K8s的Node上都会运行kube-proxy组件。在K8s集群内部，对Service Cluster IP和Port的访问可以在任意Node上进行，这是因为每个Node上的kube-proxy针对该Service都设置了相同的转发规则。
2. kube-proxy接收到Service的访问请求后，会如何选择后端的Pod呢？
首先，目前kube-proxy的负载均衡器只支持ROUNDROBIN算法。ROUNDROBIN算法按照成员列表逐个选取成员，如果一轮循环完，便从头开始下一轮，如此循环往复。kube-proxy的负载均衡器在ROUNDROBIN算法的基础上还支持Session保持。如果Service在定义中指定了Session保持，则kube-proxy接收请求时会从本地内存中查找是否存在来自该请求IP的affinityState对象，如果存在该对象，且Session没有超时，则kube-proxy将请求转向该affinityState所指向的后端Pod。如果本地存在没有来自该请求IP的affinityState对象，则按照ROUNDROBIN算法为该请求挑选一个Endpoint，并创建一个affinityState对象，记录请求的IP和指向的Endpoint。后面的请求就会黏贴到这个创建好的affinityState对象上，这就实现了客户端IP会话保持的功能。
​ kube-proxy进程为每个Service都建立了一个“服务代理对象”，服务代理对象是kube-proxy程序内部的一种数据结构，它包括一个用于监听此服务请求的SocketServer，SocketServer的端口是随机选择的一个本地空闲端口。此外，kube-proxy内部也创建了一个“负载均衡器组件”，用来实现SocketServer上收到的连接到后端多个Pod连接之间的负载均衡和会话保持能力。
​ kube-proxy通过查询和监听API Server中的Service与Endpoints的变化来实现其主要功能，包括为新创建的Service打开一个本地代理对象（代理对象是kube-proxy程序内部的一种数据结构，一个Service端口是一个代理对象，包括一个用于监听服务请求的SocketServer），接受请求，针对发生变化的Service列表，Kube-proxy会逐个处理。

外部到内部的访问
由于Service对象在Cluster IP Range池中分配到的VIP只能在内部访问，所以其他Pod都可以无障碍地访问到它。但如果这个Service作为前端服务，准备为集群外的客户端提供服务，就需要外部能够看到它。K8s支持两种对外提供服务的Service的Type定义：NodePort和LoadBalancer。


容器关联容器pause容器作用
由于Pod中所有容器共有一个网络栈，在创建Pod时候，如果该Pod没有容器或Pause容器（“kubernetes/pause”镜像创建的容器）没有启动，则先停止Pod里所有容器的进程。如果在Pod中有需要删除的容器，则删除这些容器。
用“kubernetes/pause”镜像为每个Pod创建一个容器。该Pause容器用于接管Pod中所有其他容器的网络。每创建一个新的Pod，Kubelet都会先创建一个Pause容器，然后创建其他容器。“kubernetes/pause”镜像大概为200KB，是一个非常小的容器镜像。pause容器接收所有端口访问，重定向流量到具体的各个容器中。
```

对于每个服务，我们可以通过describe 查看相应的组件的详细信息，及发生的Events。

如查看pod的所有变化事件

```shell
kubectl describe pod redis-master
```






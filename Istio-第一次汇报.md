# **Istio**

[TOC]

## Istio简介
Istio是一个与Kubernetes紧密结合的适用于云原生场景的Service Mesh形态的用于服务治理的平台。 
### 主要功能

* 流量控制（负载均衡、熔断、故障注入、重试、重定向等服务治理功能）
* 服务发现
* 可观察性（动态获取服务运行数据和输出，提供调用链、监控和调用日志收集输出可视化）
* 服务身份标识和安全
* 策略执行（通过可动态插拔、可扩展的策略实现访问控制、速率限制、配额管理、服务计费等能力）

### 架构

<center>![istio-架构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-jiagou-1.png)</center>

1. 数据平面数据平面由一组智能代理（Envoy）组成，被部署为sidecar。这些代理通过一个通用的策略和遥测中心（Mixer）传递和控制微服务之间的所有网络通信。

2. 控制平面包括Mixer、Pilot、Calley、Citadel。控制平面管理并配置代理来进行流量路由。此外，控制平面配置 Mixer 来执行策略和收集遥测数据。

## **Istio组件介绍**
### **Envoy**

<center>![Sidecar通信](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/envoy.png)</center>

#### **功能**

Envoy代理作为一个独立的进程在Istio中被部署为服务的sidecar，在逻辑上隔离服务和其他服务的通信，而这一过程，服务完全无感。

Istio 使用 Envoy 代理的扩展版本。Envoy是用C++开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。  

Envoy可以兼容多种编程语言的应用服务：在由多种语言编写的应用组成的网格中，Envoy作为桥梁，连接不同编程语言的应用。

这种 sidecar 部署允许 Istio提取大量关于流量行为的信号作为属性，istio将这些属性发送给组件Mixer，Istio 可以在 Mixer 中使用这些属性来执行决策，并将它们发送到监控系统，以提供整个网格的行为信息。

>Istio使用```属性```来控制在服务网格中运行的服务的运行时行为。属性是具有名称和类型的元数据片段，用以描述入口和出口流量，以及这些流量所属的环境。Istio属性携带特定信息片段，例如API请求的错误代码，API请求的延迟或TCP连接的原始IP地址。

sidecar初始化方式是非常方便的，不需要对原有应用架构进行重新设计。

由 Envoy 代理启用的一些 Istio 的功能和任务包括:

* 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
* 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
* 安全性和身份验证特性：执行安全性策略以及通过配置API定义的访问控制和速率限制。

#### **Envoy内存占用问题**

描述:
一个Envoy进程包括一个Server线程和一个GuardDog守护线程，其中Server线程负责管理Access Log及解析上游主机的DNS等，GuardDog负责看门狗业务。除了这两个功能固定的线程外，一个Envoy进程配置多个Listener，每个Listener都独立调度，每个Listener下都创建多个线程，每个线程对应一个worker，多个worker并行处理该listener的事务

* 这就导致每个Envoy为了监听来自其他服务的信息创建了大量Listener，这大量占据了内存。

* Envoy内存大小和配置相关，和请求处理速率无关。内存占用和Listener、Cluster的数量成线性相关

* 一个微服务不需要和其他所有的微服务通信，大量的没用Listener占用了大量的内存。

解决方案

* 按namespace对配置进行隔离。istio 1.3 之后缺省用namespace对Service进行隔离。问题：一个namespace下微服务仍然太多，效果不好，仍然存在无用Listener。

* 按服务访问关系进行细粒度隔离。Envoy中只配置与该Sidecar代理服务所需访问的外部服务的相关配置，可有效降低内存使用程度。  

详细信息请点击[这里](https://www.servicemesher.com/blog/201911-envoy-memory-optimize/)

### <b>Mixer</b>

<center>![](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-mixer-1.png)</center>

#### **Mixer架构**
在服务间进行请求转发时，Envoy对Mixer发去Check、Report这两次请求，即在转发请求前Mixer执行访问策略与管理配额。并在请求转发后上报遥测数据。Mixer的访问策略执行、配额管理、遥测数据收集都是基于属性的，Envoy会将每次请求的信息放到属性集中，再将属性集发送到Mixer执行相应的处理。
#### **功能**

1. 前提条件检查：在响应服务调用者的请求之前，根据前提条件检查提前验证。前置条件包括服务间的认证、黑白名单、是否通过ACL检查，等等。

2. 配额管理：能够在多个维度分配和释放配额。

3. 遥测报告：使服务能够上报日志和进行监控。

#### **服务组件**
istio控制平面部署了两个Mixer组件：istio-telemetry、istio-policy。

* istio-telemetry：专门用于收集遥测数据的Mixer服务组件。当网格中的两个服务间有调用发生时，服务的代理Envoy就会上报遥测数据给istio-telemetry服务组件，istio-telemetry服务组件根据配置将生成访问Metric等数据分发给后端的遥测服务。数据面通过Report接口上报数据时访问数据会被批量上报。

* istio-policy：数据面在转发服务的请求前调用istio-policy的Check接口检查是否允许访问，Mixer根据配置将请求转发到对应的Adapter做对应检查，给代理返回允许访问还是拒绝。可以对接如配额、授权、黑白名单等不同的控制后端，对服务间的访问进行可扩展的控制。

#### **性能问题**  

过程：

`Envoy`每次收到请求（记为A），都需要从其中提取一组属性，发起两次对`Mixer`的请求。

* 处理A请求之前：这时需要做前提条件检查和配额管理，envoy将A中相应的属性远程传给Mixer，Mixer中的adapter会对这些属性进行处理，只有满足条件的请求才会被转发。

* 在处理并转发A请求之后：这时要向Mixer上报遥测报告（日志）等。

* Mixer独立出`out-of-process adapter`,将适配器独立于Mixer，除了上述两次访问之外，Mixer还要对`adapter`进行额外的远程访问



#### **改良方案**

* 针对遥测报告的处理：进行异步处理，对遥测报告进行缓存，一定时间后一起上传。

* 针对前提条件检查和配额管理：在`Envoy`端建立`Mixer Cathe`，建立缓存机制。但是由于在Envoy端缓存和缓存逻辑相分离，只要是之前上传过得请求属性组合都会被缓存下来，由于属性值的组合方式太多，笛卡尔乘积太大，导致该方案非常不友好。

* 由于`Mixer Cathe`方案不友好，正在推进`Mixer v2`，也就是`去Mixer`化，将Mixer功能集合到`Envoy`中。但这种改进与istio设计理念相背离，进展很慢。

* 目前Istio安装可以通过自定义设置禁用Mixer。

``` 
     istioctl manifest apply --set telemetry.enabled=false --set policy.enabled=false   
```



##### **业内成熟解决方案** 

蚂蚁金服彻底放弃`Mxier`组件，将功能集合到`Envoy（MOSN）`中，在`sidecar`上完成前提条件检查、配额管理、遥测报告等功能。

### <b>Pilot</b>

#### **功能**

1. 服务网格间的流量管理。

2. 控制面板和数据面的配置下发。

Pilot是控制流量管理的核心组件，管理和配置部署在istio服务网格中的所有Envoy代理实例，允许用户创建Envoy代理之间的流量转发路由规则，并配置故障恢复功能，例如超时、重试及熔断。Pilot维护着网格中所有的服务实例信息，并基于xDS服务发现让每个Envoy都能了解上游服务的实例信息。

#### **架构**

Pilot架构主要分为三层

<center>![](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-pilot-3.png)</center>

* 平台适配器。平台适配器负责监听底层平台，并完成从平台特有的服务模型到istio服务模型的转换。
    * 服务模型转换：将Kubernetes、Consul等平台的服务模型转换为istio规范的服务模型
    
    * 服务实例转换：将Kubernet EndPoint资源转换为Istio规范的服务实例模型
    
    * Istio中配置模型的转换：比如将Kubernetes平台非结构化的Custom  Resources配置规则转换为VirtualService、Gateway、ServiceEntry、DestinationRule等API，以及将Kubernetes Ingress资源转换为Istio Gateway资源。
* 抽象聚合层。Pilot支持基于多个不同的底层平台进行服务发现和流量规则发现。抽象聚合层通过聚合不同平台的服务、配置规则对外提供统一的接口，进而使得Pilot发现服务（xDS）无须关心底层平台的差异，达到解耦xDS与底层平台的目的。

* xDS发现服务：将Pilot流量治理的能力暴露给客户端。Pilot通过xDS服务器提供服务发现接口xDS API，xDS服务器接收并维护Envoy代理的连接，并基于客户端订阅的资源名称进行相应xDS配置的分发。

Pilot和每一个Envoy之间都维护着一条个gRPC长连接，所有配置下发都通过这条连接。配置触发条件为：底层服务注册中心发生变化、规则配置更新、Envoy主动发起更新请求。

pilot接收两个数据来源

* 服务数据：来源于服务注册表(`Service Registry`)，例如`Kubernetes`中注册的`Service`。`pilot`本身不进行服务发现，需要基于平台(例如K8S)的服务注册表。Istio将相关信息标准化，提供给Envoy。

* 配置规则：路由规则、流量管理规则等，通过`Kubernetes CRD(Custom Resources Definition)`形式定义并存储在Kubernetes中。  

该方式和Kubernetes耦合严重，所以提出了`MCP(Mesh Configuration Protocol)`协议，`Istio Pilot`作为`MCP Client`，任何实现了`MCP`协议的Server都可以通过MCP协议向Pilot下发配置，从而解除了istio和Kubernetes的耦合。



pilot 输出符合xDS接口的数据面配置数据，并通过gRPC Streaming接口将配置数据推送到数据面的Envoy中。

#### **标准数据面API(_Envoy V2 API_)**  



该[Envoy V2 API](https://github.com/envoyproxy/data-plane-api/blob/master/API_OVERVIEW.md)为`Istio`和`Envoy`项目联合制定，作为Istio控制面和数据面流量管理的标准接口。  

`Pilot`使用了该[标准数据面API](https://github.com/envoyproxy/data-plane-api/blob/master/API_OVERVIEW.md)来将服务信息和流量规则下发到数据面的`sidecar`中。该API有多种接口(`xDS`)。 

通过该`API`，`istio`将控制面和数据面进行了解耦，为多种`Sidecar`实现提供了可能。目前基于该API已经实现了多种`Sidecar`与`istio`的集成，也可以通过该API自己编写Sidecar实现



#### **主要结构**
<center>![istio结构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-pilot-1.png)</center>

`istio-pilot`分为`istio-proxy(sidecar)`和`discovery`两个容器。其中`sitio-proxy`在数据面负责`Envoy`的生命周期，`discovery`负责流量控制、配置下发等主要功能，后者主要分为三个部分

1. `Config Controller`

2. `Service Controller`

3. `Envoy Xds Server`  



##### Config Controller

`Config Controller`用于管理各种配置数据，包括用户创建的流量管理规则和策略。目前支持三种类型的Config Controller：

* `Kubernetes`：使用Kubernetes作为配置数据的存储，依附于Kubernetes强大的CRD机制。

* `MCP(Mesh Configuration Protocol)`：解决与K8S的耦合。实现了MCP协议的Server都可以通过MCP协议给Pilot下发配置。

* `Memory`：一个在内存中的Config Controller实现，主要用于测试。 



目前`Istio`的配置包括：

* `Virtual Service`：定义流量路由规则。

* `Destination Rule`：定义和一个服务或者subset相关的流量处理规则，包括负载均衡策略，连接池大小，断路器设置，subset定义等。

* `Gateway`：定义入口网关上对外暴露的服务。

* `Service Entry`：通过定义一个Service Entry可以将一个外部服务手动添加到服务网格中。

* `Envoy Filter`：通过Pilot在Envoy的配置中添加一个自定义的Filter。

##### Service Controller

`Service Controller`用于管理各种`Service Registry`，得到服务注册表数据。目前支持的Service Registry包括：

* `Kubernetes`：对接`Kubernetes Registry`,将Kubernetes中定义的`Service`和`Instance`采集到istio中。

* `Consul`：对接`Consul Catalog`

* `MCP`： （解决耦合。实现该协议的`Server Registry`都可以获取到相应的`Service`和`Service Instance`）

* `Memory`： 用于测试的内存中`Service Controller`实现。

##### EnvoyXdsService

主要功能逻辑：

* 启动`gRPC Server`并接收来自Envoy端的连接请求。

* 接收`Envoy`端`xDS`请求，从`Config Controller`和`Service Controller`中获取配置和服务信息，生成响应信息发送给Envoy。

* 监听来自`Config Controller`的配置变化信息和来自`Service Controller`的服务变化信息，将配置和服务变化内容通过`xDS`推送到`Envoy`。

#### **性能问题**

Pilot配置下发采取的是全量分发模式，每一个配置发生变化，都需要进行全量更新，这回浪费大量的资源。

改良机制：  

* 多级缓存

* 去抖动开发

* EDS增量分发模式

* 资源隔离

* 序列化优化

* 预计算优化

##### 三级缓存优化

1. 平台适配层缓存。以`K8S`为例，所有服务及配置规则的监听都通过`K8S Informer`实现。`Informer`的`LIST-WATCH`原理是通过在客户端本地维护资源的缓存实现的。这是第一级缓存。

2. 中间层（`Istio API`模型）层缓存。平台层的资源（`Service、Endpoint、VirtualService、DestinationRule`等）都是原始的API模型，对于具体`Sidecar`、`gateway`配置规则的生成还需要平台层资源的选择和到`Istio`资源模型的转换。如果在资源模型转换中重复执行，会影响性能。所以istio在中间层做了`Istio`资源模型的缓存优化。

3. `xDS`配置缓存。`xDS`层面有两种配置缓存：`CLuster`和`Endpoint`，这两种资源较为通用，因此针对两者进行缓存。

##### 去抖动开发

1. 最小静默时间：记为`minQuiet`。`t`<sub>`n`</sub>表示在一个推送周期内第`N`次接收到更新事件的时间，如果在接下来的`minQuiet`时间内没有新的更新事件发生，那么在`t`<sub>`n`</sub>`+minQuiet`时间点分发时间到`PushChannel`。

2. 最大延迟：如果在一个推送周期内一直有更新发生，且间隔都小于`minQuiet`，则更新一直无法分发，会造成极大地延迟。于是设置最大延迟时间，到达该时间则直接进行分发。

##### 增量EDS

服务实例（`Endpoint`）数量最大、变化最快。仅由于endpoint的更细引发全量xDS配置分发，会浪费很多计算资源与网络带宽资源，同时影响Envoy代理的稳定性。  

做法：在一个推送周期内，`EnvoyXdsServer`会缓存所有`IstioEndpoint`信息和发生变化的服务列表。当`K8S`内`Endpoint`发生更新时，先将其转换为`istioEndpoint`,并与`EnvoyXdsServer`中缓存作对比。服务周期结束时，如果只发生了`EndPoint`的更新变化，则进行EDS增量分发，否则进行全量分发。

##### 资源隔离

利用命名空间隔离的概念，有效定义访问范围和服务的有效作用范围。  

目前做了两方面的优化：

1. 用`Sidecar API`资源定义`Envoy`代理可以访问的服务

2. 用服务及配置（`VirtualService、DestinationRule`）资源定义其有效范围   

#### **业务DSL语言**

`Pilot`定义了一套`DSL(Domain Specific Language)`语言，提供了面向业务的高层抽象，可以提供给运维人员理解和使用。`DSL`是基于`K8S API Server` 中的`Custom Resource (CRD)` 实现，使用与创建方法与K8S资源类(例如`Service Pod Deployment`)类似。  

运维人员使用`DSL`定义流量规则并下发到`Pilot`，这些规则在`Pilot`翻译成数据面的配置，在通过`标准API`分发到`Envoy`实例，可以在运行期对微服务流量进行控制调整。

### **Galley**
**描述：**    

Istio的配置验证、引入、处理和分发组件。统一管理配置。负责将istio组件的其余部分与从底层平台获取到的用户配置的详细信息隔离。Galley使用MCP(Mesh Configuration Protocol)协议和其他组件进行配置的交互。

#### **MCP协议**

MCP是在Istio网格中定义的传输协议，用来在网格内的组件和Galley之间传输配置信息，基于gRPC协议。Galley作为服务器端，接收来自Pilot、Mixer等组件的请求，各组件通过MCP从Galley中获取用户的配置信息，从而解耦网格内的各组件与底层平台。  

一次完整的MCP请求包括请求、响应和ACK/NACK，首先，各组件主动向Galley发起MeshConfigRequest，然后Galley根据请求返回相应的MeshConfigResponse，最后，各组件在接收到配置信息后进行动态加载，若加载成功，进行ACK，否则进行NACK。

#### **配置聚合和分发**
Galley实现了组件配置信息的统一聚合与分发，隔离了网格内的组件与底层数据平台，使各组件无需感知底层数据平台的差异，统一从Galley中获取配置信息， 从而更加聚焦于各组件的自身业务。

##### 配置聚合
* 配置聚合将从底层平台获取的用户配置信息保存到本地缓存中，通过自定义Source接口，将底层平台的用户事件保存到缓存队列Events中。

* 在事件进入Events中后，Galley将事件存入State对象，State对象的entries属性根据配置信息的类型来存储配置。例如，将所有VirtualService类型的数据都保存在一个对象中，在组件请求VirtualService类型资源时可将数据统一返回。

* 数据在State中缓存到一定条件时，会触发将在State中缓存的数据已Snapshot的形式保存到Cathe，Cathe会为用户请求返回完整、稳定的数据。只有将K8S中的配置数据全部获取完毕后才会响应组件的请求并返回数据，否则会造成配置不一致的情况。

##### 配置分发
* 被动分发：接收gRPC请求，返回相应数据。

* 主动分发：由底层数据平台事件触发。

### **Citadel**  

Citadel负责证书颁发和轮换。Istio默认提供的双向TLS安全功能就是依赖Citadel签发的证书进行TLS认证的。
Citadel作为网格内唯一的身份管理组件，主要负责为集群内的服务账户颁发证书、启动gRPC服务器以及证书轮换器等组件


## **Sidecar注入**

Istio的流量管理、策略、遥测等功能不需要应用程序做任何改动，这种无侵入式的方式依赖于Sidecar。
Sidecar与应用容器共存于同一个Pod中，并且共享同一个Network Namespace，因此Sidecar容器与应用容器共享同一个网络协议栈。应用程序发送或者接收的所有流量通过iptables都被Sidecar拦截，并由Sidecar进行认证、鉴权、策略执行及。遥测数据上报等众多治理功能。

Sidecar注入容器有两种方式，手动注入和自动注入。

1. 手动注入：修改pod资源定义，添加两个新容器

    * 容器`istio-init`，配置iptables,劫持流量

    * 容器`istio-proxy`, 两个进程`pilot-agent`和`envoy`,  `pilot-proxy`初始化`envoy`
2. 自动注入：
* Sidecar-injector是istio中实现自动注入Sidecar的组件，它是以Kubernetes准入控制器Admission Controller的形式运行的。Admission Controller的基本工作原理是拦截Kube-apiserver的请求，在对象持久化之前、认证鉴权之后进行拦截。Kubernetes允许用户以Webhook的方式自定义准入控制器，Sidecar injector就是一种特殊的MutatingAdmissionWebhook。
* Sidecar injector在创建Pod时进行Sidecar容器注入，在Pod的创建请求到达Kube-apiserver后，首先进行认证鉴权，然后在准入控制阶段，kube-apiserver以REST的方式同步调用Sidecar injector Webhook服务进行istio-init和istio-proxy容器的注入，最后将对象持久化到etcd中。

## **流量治理**

* Virtual Service：定义流量路由规则。

* Destination Rule：定义和一个服务或者subset相关的流量处理规则，包括负载均衡策略，连接池大小，断路器设置，subset定义等。

* Gateway：定义入口网关上对外暴露的服务。

* Service Entry：通过定义一个Service Entry可以将一个外部服务手动添加到服务网格中。

* Envoy Filter：通过Pilot在Envoy的配置中添加一个自定义的Filter。



## **服务发现**

Istio的服务发现机制是基于底层平台的服务注册表的。
```Service Registry```，代表一个通过适配器插入到Pilot中的服务注册表，即Kubernetes，Cloud Foundry 或者 Consul 等具体后端的服务部署/服务注册发现平台。
```Service Controller```接口是Pilot组件的一个服务。该接口会一直监听Registry的变化，当Registry中```Service```、```ServiceInstance```中发生变化时，会将变化通知到Controller中的Service Handler中。
```Service Discovery```接口，该接口可以获取到Registry中具体的Service、ServiceInstance。
当Service或ServiceInstance发生变化时，Pilot通过上述服务得到变化后的Registry信息，并将新的服务注册信息通过```xDS```接口发送给Envoy。



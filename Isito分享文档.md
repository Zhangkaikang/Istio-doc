# Isito分享文档

[TOC]

## Istio源起

### Service Mesh

`Service Mesh` 又译作服务网格，是致力于解决服务间通讯的基础设施层。它负责通过包含现代云原生应用程序的复杂服务拓扑来可靠地传递请求。实际上，服务网格通常通过一组轻量级网络代理（`Sidecar`）来实现，这些代理和应用程序代码一起部署，而不需要感知应用程序本身。

简而言之，服务网格是服务（包括）微服务之间通信的控制器。随着越来越多的容器应用的开发和部署，服务会越来越多，维护应用之间的通信、服务间的负载均衡、流量管理、路由、运行状况监视、安全策略和服务间身份认证变得越来越困难。为了解决这个问题，服务网格应运而生。服务网格会为每一个服务容器提供一个相应的网络代理（Sidecar）容器，该容器接管服务容器需要的网络通信，从而让应用程序和网络通信解耦。

服务网格有如下几个特点：

1. 应用程序间通讯的中间层
2. 轻量级网络代理
3. 应用程序无感知
4. 解耦应用程序的重试/超时、监控、追踪和服务发现

### Istio是什么

Istio是一个与Kubernetes紧密结合的适用于云原生场景的Service Mesh形态的用于服务治理的开放平台。

#### 基于Kubernetes

Kubernetes作为容器平台标准，得到了广泛的应用。Kubernetes本身已经提供了足够强大的功能，例如应用负载的部署、升级、扩容等运行管理能力。其中的Service机制可以做服务注册、服务发现和负载均衡功能，同时支持通过服务名访问到服务实例。

Kubernetes本身支持微服务架构，在Pod中部署微服务是一种很好的选择，但是没有解决服务间访问的管理问题，例如：服务的熔断、限流、动态路由、调用链追踪。Istio可以完美的解决这些问题。

#### Istio

Istio是部署在Kubernetes平台上的服务网格，最大化的利用了K8S这个基础设施。它通过K8S自带的服务注册、服务发现功能，将K8S中的`Service`和`Endpoint`转换为Istio服务模型的`Service`和`Service Instance`。

Istio的路由规则和使用策略通过K8S CRD实现，存储在Kube-apiserver中。

Istio功能：

* 连接（通过控制流量规则）
  * 负载均衡
  * 熔断
  * 故障注入
  * 重试
  * 重定向
* 安全
  * 透明认证机制
  * 通道机密
  * 服务访问授权
* 策略执行（动态插拔、可扩展）
  * 访问控制
  * 速率限制
  * 配额管理
  * 服务计费
* 可观察性
  * 提供调用链、监控和调用日志收集输出



## Istio架构

### 服务网格的衍变

### Isito架构

Istio架构总体上分为数据平面和控制平面两种。

<center>![Isito架构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-jiagou.png)</center>

* 数据平面由一组智能代理（Envoy）组成，被部署为Sidecar。这些代理通过一个通用的策略和遥测中心（Mixer）传递和控制微服务之间的所有网络通信。
* 控制平面包括Mixer、Pilot、Calley、Citadel。控制平面管理并配置代理来进行流量路由。此外，控制平面配置 Mixer 来执行策略和收集遥测数据。

> 在本篇文章中，Sidecar、Envoy、Proxy等同，指向的都是数据平面中和业务容器部署在一起的网络代理容器，Isito中使用的Sidecar为Envoy。

#### 数据平面

<center>![](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/envoy.png)</center>

数据面Sidecar运行在Kubernetes中的Pod中，作为一个Proxy和业务容器部署在一起。

##### Envoy

Envoy代理作为一个独立的进程在Istio中被部署为服务的sidecar，在逻辑上隔离服务和其他服务的通信，而这一过程，服务完全无感。

Istio 使用 Envoy 代理的扩展版本。Envoy是用C++开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。  

Envoy可以兼容多种编程语言的应用服务：在由多种语言编写的应用组成的网格中，Envoy作为桥梁，连接不同编程语言的应用。

这种 sidecar 部署允许 Istio提取大量关于流量行为的信号作为属性，istio将这些属性发送给组件Mixer，Istio 可以在 Mixer 中使用这些属性来执行决策，并将它们发送到监控系统，以提供整个网格的行为信息。

>Istio使用```属性```来控制在服务网格中运行的服务的运行时行为。属性是具有名称和类型的元数据片段，用以描述入口和出口流量，以及这些流量所属的环境。Istio属性携带特定信息片段，例如API请求的错误代码，API请求的延迟或TCP连接的原始IP地址等。

由 Envoy 代理启用的一些 Istio 的功能和任务包括:

* 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
* 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
* 安全性和身份验证特性：执行安全性策略以及通过配置API定义的访问控制和速率限制。

#### 控制平面

##### Pilot

###### 功能

* 服务网格间的流量管理
* 控制平面和数据面的配置下发

Pilot是控制流量管理的核心组件，管理和配置部署在istio服务网格中的所有Envoy代理实例，允许用户创建Envoy代理之间的流量转发路由规则，并配置故障恢复功能，例如超时、重试及熔断。Pilot维护着网格中所有的服务实例信息，并基于xDS服务发现让每个Envoy都能了解上游服务的实例信息。

###### 架构

Pilot架构主要分为三层

<center>![Pilot架构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-pilot-3.png)</center>

* 平台适配器。平台适配器负责监听底层平台，并完成从平台特有的服务模型到istio服务模型的转换。
  * 服务模型转换：将Kubernetes、Consul等平台的服务模型转换为istio规范的服务模型

  * 服务实例转换：将Kubernet EndPoint资源转换为Istio规范的服务实例模型

  * Istio中配置模型的转换：比如将Kubernetes平台非结构化的Custom  Resources配置规则转换为VirtualService、Gateway、ServiceEntry、DestinationRule等API，以及将Kubernetes Ingress资源转换为Istio Gateway资源。
* 抽象聚合层。Pilot支持基于多个不同的底层平台进行服务发现和流量规则发现。抽象聚合层通过聚合不同平台的服务、配置规则，对外提供统一的接口，进而使得Pilot发现服务（xDS）无须关心底层平台的差异，达到解耦xDS与底层平台的目的。
* xDS发现服务：将Pilot流量治理的能力暴露给客户端。Pilot通过xDS服务器提供服务发现接口xDS API，xDS服务器接收并维护Envoy代理的连接，并基于客户端订阅的资源名称进行相应xDS配置的分发。

Pilot和每一个Envoy之间都维护着一条个gRPC长连接，所有配置下发都通过这条连接。配置触发条件为：底层服务注册中心发生变化、规则配置更新、Envoy主动发起更新请求。

pilot接收两个数据来源

* 服务数据：来源于服务注册表(`Service Registry`)，例如`Kubernetes`中注册的`Service`。`pilot`本身不进行服务发现，需要基于平台(例如K8S)的服务注册表。Istio将相关信息标准化，提供给Envoy。
* 配置规则：路由规则、流量管理规则等，通过`Kubernetes CRD(Custom Resources Definition)`形式定义并存储在Kubernetes中。  

###### 主要结构

<center>![istio结构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-pilot-1.png)</center>
`istio-pilot`分为`istio-proxy(sidecar)`和`discovery`两个容器。其中`sitio-proxy`在数据面负责`Envoy`的生命周期，`discovery`负责流量控制、配置下发等主要功能，后者主要分为三个部分

1. `Config Controller`
2. `Service Controller`
3. `Envoy Xds Server`  

###### Config Controller

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

###### Service Controller

`Service Controller`用于管理各种`Service Registry`，得到服务注册表数据。目前支持的Service Registry包括：

* `Kubernetes`：对接`Kubernetes Registry`,将Kubernetes中定义的`Service`和`Instance`采集到istio中。

* `Consul`：对接`Consul Catalog`

* `MCP`： （解决耦合。实现该协议的`Server Registry`都可以获取到相应的`Service`和`Service Instance`）

* `Memory`： 用于测试的内存中`Service Controller`实现。

###### EnvoyXdsService

主要功能逻辑：

* 启动`gRPC Server`并接收来自Envoy端的连接请求。
* 接收`Envoy`端`xDS`请求，从`Config Controller`和`Service Controller`中获取配置和服务信息，生成响应信息发送给Envoy。
* 监听来自`Config Controller`的配置变化信息和来自`Service Controller`的服务变化信息，将配置和服务变化内容通过`xDS`推送到`Envoy`。

###### 业务DSL语言

`Pilot`定义了一套`DSL(Domain Specific Language)`语言，提供了面向业务的高层抽象，可以提供给运维人员理解和使用。`DSL`是基于`K8S API Server` 中的`Custom Resource (CRD)` 实现，使用与创建方法与K8S资源类(例如`Service Pod Deployment`)类似。  

运维人员使用`DSL`定义流量规则并下发到`Pilot`，这些规则在`Pilot`翻译成数据面的配置，在通过`标准API`分发到`Envoy`实例，可以在运行期对微服务流量进行控制调整。

##### Mixer

<center>![Mixer架构](https://raw.githubusercontent.com/Zhangkaikang/md-pic/master/istio-mixer-1.png)</center>

在服务间进行请求转发时，Envoy对Mixer发起Check、Report这两次请求，即在转发请求前Mixer执行访问策略与管理配额。并在请求转发后上报遥测数据。Mixer的访问策略执行、配额管理、遥测数据收集都是基于属性的，Envoy会将每次请求的信息放到属性集中，再将属性集发送到Mixer执行相应的处理。

###### 功能

1. 前提条件检查：在响应服务调用者的请求之前，根据前提条件检查提前验证。前置条件包括服务间的认证、黑白名单、是否通过ACL检查，等等。
2. 配额管理：能够在多个维度分配和释放配额。
3. 遥测报告：使服务能够上报日志和进行监控。

###### 服务组件

istio控制平面部署了两个Mixer组件：istio-telemetry、istio-policy。

* istio-telemetry：专门用于收集遥测数据的Mixer服务组件。当网格中的两个服务间有调用发生时，服务的代理Envoy就会上报遥测数据给istio-telemetry服务组件，istio-telemetry服务组件根据配置将生成访问Metric等数据分发给后端的遥测服务。数据面通过Report接口上报数据时访问数据会被批量上报。

* istio-policy：数据面在转发服务的请求前调用istio-policy的Check接口检查是否允许访问，Mixer根据配置将请求转发到对应的Adapter做对应检查，给代理返回允许访问还是拒绝。可以对接如配额、授权、黑白名单等不同的控制后端，对服务间的访问进行可扩展的控制。

##### Galley

**描述：**    

Istio的配置验证、引入、处理和分发组件。统一管理配置。负责将istio组件的其余部分与从底层平台获取到的用户配置的详细信息隔离。Galley使用MCP(Mesh Configuration Protocol)协议和其他组件进行配置的交互。

###### MCP协议

MCP是在Istio网格中定义的传输协议，用来在网格内的组件和Galley之间传输配置信息，基于gRPC协议。Galley作为服务器端，接收来自Pilot、Mixer等组件的请求，各组件通过MCP从Galley中获取用户的配置信息，从而解耦网格内的各组件与底层平台。  

一次完整的MCP请求包括请求、响应和ACK/NACK，首先，各组件主动向Galley发起MeshConfigRequest，然后Galley根据请求返回相应的MeshConfigResponse，最后，各组件在接收到配置信息后进行动态加载，若加载成功，进行ACK，否则进行NACK。

###### 配置聚合和分发

Galley实现了组件配置信息的统一聚合与分发，隔离了网格内的组件与底层数据平台，使各组件无需感知底层数据平台的差异，统一从Galley中获取配置信息， 从而更加聚焦于各组件的自身业务。

###### 配置聚合

* 配置聚合将从底层平台获取的用户配置信息保存到本地缓存中，通过自定义Source接口，将底层平台的用户事件保存到缓存队列Events中。

* 在事件进入Events中后，Galley将事件存入State对象，State对象的entries属性根据配置信息的类型来存储配置。例如，将所有VirtualService类型的数据都保存在一个对象中，在组件请求VirtualService类型资源时可将数据统一返回。

* 数据在State中缓存到一定条件时，会触发将在State中缓存的数据已Snapshot的形式保存到Cathe，Cathe会为用户请求返回完整、稳定的数据。只有将K8S中的配置数据全部获取完毕后才会响应组件的请求并返回数据，否则会造成配置不一致的情况。

###### 配置分发

* 被动分发：接收gRPC请求，返回相应数据。

* 主动分发：由底层数据平台事件触发。

##### Citadel

Citadel负责证书颁发和轮换。Istio默认提供的双向TLS安全功能就是依赖Citadel签发的证书进行TLS认证的。
Citadel作为网格内唯一的身份管理组件，主要负责为集群内的服务账户颁发证书、启动gRPC服务器以及证书轮换器等组件

### 工作流程（简略）

1. 自动注入：在创建应用程序时自动注入Sidecar代理。在K8S创建Pod时，Kube-apiserver调用管理面组件的Sidecar-Injector服务，自动修改应用程序的描述信息并注入Sidecar。在Pod创建时，业务容器和Sidecar同时创建。
2. 流量拦截：在Pod初始化时设置iptables规则，当有流量到来时，基于配置的iptables规则拦截业务容器的Inbound流量和Outbound流量到Sidecar上，业务容器对此无感，仍然按照原有方式进行互相访问。
3. 服务发现：服务发起方的Envoy调用Pilot的服务发现接口获取目标服务的实例列表。
4. 负载均衡：服务发起方的Envoy根据配置的负载均衡策略选择服务实例，并连接对应的实例地址。
5. 流量治理：Envoy从Pilot中获取配置的流量规则，在拦截到Inbound流量和Outbound流量时进行治理逻辑。例如将不同特征的流量分发到不同版本的服务。
6. 访问安全：在服务间访问时通过双方的Envoy进行双向认证和通道加密，并基于服务的身份进行授权管理。
7. 服务遥测：在服务间通信时，通信双方的Envoy都会连接管理面组件Mixer上报访问数据，并通过Mixer将数据转发给对应的监控后端。
8. 策略执行：在进行服务访问时，通过Mixer连接后端服务来控制服务间的访问，判断访问是放行还是拒绝。
9. 外部访问：在网格的入口处有一个Envoy扮演入口网关（Gateway）的角色。

## Istio性能

### Mixer性能

#### 原因

`Envoy`每次收到请求（记为A），都需要从其中提取一组属性，发起两次对`Mixer`的请求。

* 处理A请求之前：这时需要做前提条件检查和配额管理，envoy将A中相应的属性远程传给Mixer，Mixer中的adapter会对这些属性进行处理，只有满足条件的请求才会被转发。
* 在处理并转发A请求之后：这时要向Mixer上报遥测报告（日志）等。
* Mixer独立出`out-of-process adapter`,将适配器独立于Mixer，除了上述两次访问之外，Mixer还要对`adapter`进行额外的远程访问

#### 改良方案

* 针对遥测报告的处理：进行异步处理，对遥测报告进行缓存，一定时间后一起上传。

* 针对前提条件检查和配额管理：在`Envoy`端建立`Mixer Cathe`，建立缓存机制。提高命中率，减少对Mixer的调用。

* 上述方案并没有实质性解决问题，目前社区正在推进`Mixer v2`，也就是`去Mixer`化，将Mixer功能集合到`Envoy`中。但是还没有具体的成果。

* 目前Istio安装可以通过自定义设置禁用Mixer。

### Pilot性能

#### 原因

Pilot配置下发采取的是全量分发模式，每一个配置发生变化，都需要进行全量更新，这回浪费大量的资源。

#### 改良机制

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

### 性能测试中发现的问题

 性能消耗突出组件

* istio-ingressgateway：istio自带网关组件，负责相应IP地址的网关服务，对部署节点配置要求加高。流量大时，容易成为性能瓶颈，需要单独的部署到性能高的节点上，与其他控制平面、业务节点分开。
* istio-telemetry：Mixer有两个组件，分别为istio-telemetry和istio-policy。在不禁用Mixer的情况下，istio-policy性能消耗小，istio-telemetry性能消耗严重。由于每一次网格内请求都会向istio-telemetry发送遥测数据，导致该组件占用CPU过高
* istio-tracing：jaeger服务。在默认情况下跟踪百分之一的数据，当我将参数调整为百分之百时，发现他的CPU消耗跃升到所有组件中最高。

1. 部署istio时，更改控制平面组件的默认配置。指定部署节点、replicas数量、hpa伸缩条件、cpu内存限制。
2. 指定部署节点时，需要尽量将控制平面和数据平面分开。由于K8S特性，如果不限制节点选取规则，控制平面节点就会出现在任意一个节点上，不容易定位问题。
3. 将消耗性能严重的组件安装在性能高的节点上，不要让他轻易的成为性能瓶颈。




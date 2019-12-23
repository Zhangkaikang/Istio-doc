# 测试机安装Istio及压测
标签（空格分隔）： 未分类

---

[TOC]
## **安装前问题**
   * istio-system隔离，控制组件安装在该命名空间中
   * istio-sidecar-injector，自动注入Sidecar隔离。将自动注入限制在某一命名空间中，或者限制在某一个Deployment中。注意：Sidecar隔离的应用与非Sidecar应用通信时可能出现问题。
   * 网关服务、控制平面服务应该限制在几个高性能节点上，数据平面、业务服务限制在自己的节点。
   * 业务服务、监测服务应该使用不同的loadBalancer

## **安装Istio**
### 下载Istio
```shell
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.4.1
//将bin目录下的istioctl工具加进环境变量
$ export PATH=$PWD/bin:$PATH
```

### **安装Istio**
#### **安装**
自定义安装，在Default安装的基础上，安装grafana组件
默认安装组件如下：

       * istio-citadel
       * istio-galley
       * istio-ingressgateway
       * istio-pilot 
       * istio-sidecar-injector
       * istio-policy
       * istio-telemetry
       * prometheus
```shell
$ istioctl manifest apply --set values.grafana.enabled=true --set values.gateways.enabled=true 
```
#### **命名空间创建**
创建命名空间istio-test 
```shell
$ kubectl create namespace istio-test
```
为命名空间istio-test加入Sidecar自动注入
```shell
$ kubectl label namespace istio-test istio-injection=enabled
```
不为命名空间自动注入，只注入某个Deployment时
```shell
$ istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
```
### **修改默认配置**
#### **节点label**
```shell
//更改node配置下metadata.labels.node=<your-node-name>
$ kubectl get nodes 
$ kubectl edit node <your-node-name> 
```
#### **ingressgateway**
修改hpa中的MinReplicas、MaxReplicas、targetCPUUtilizationPercentage
```shell
$ kubectl get hpa -n istio-system
$ kubectl edit hpa istio-ingressgateway -n istio-system
```
修改Deployment部署,在spec.template.spec下创建nodeSelector,值为目的部署节点的label。
找到resources.limits,限制CPU、Memory使用
```shell
$ kubectl get deployment -n istio-system
$ kubectl edit deployment istio-ingressgateway -n istio-system
```
```yaml
nodeSelector:
  node: <your-node-label>
  
 resources:
   limits:
     cpu: "2"
     memory: 1Gi
    requests:
      cpu: 100m
      memory: 128Mi

```
#### **istio-telemetry**
修改hpa中的MinReplicas、MaxReplicas、targetCPUUtilizationPercentage
```shell
$ kubectl get hpa -n istio-system
$ kubectl edit hpa istio-telemetry -n istio-system
```
修改Deployment部署,在spec.template.spec下创建nodeSelector,值为目的部署节点的label
找到resources.limits,限制CPU、Memory使用

```shell
$ kubectl get deployment -n istio-system
$ kubectl edit deployment istio-ingressgateway -n istio-system
```
```shell
nodeSelector:
  node: <your-node-label>
  
 resources:
   limits:
     cpu: "2"
     memory: 1Gi
    requests:
      cpu: 100m
      memory: 128Mi

```
#### **其他控制节点**
如有必要，应该像上述服务一样，更改相应配置文件

## **安装服务**
### **ingressgateway2**
创建新的网关LoadBalancer，配置ingressgateway2负载节点。该服务部署在istio-system命名空间。
```shell
$ kubectl apply -f ingressgateway2.yaml
$ kubectl edit deployment ingressgateway2 -n istio-system
```
### **Prometheus、Grafana**
创建Prometheus相关Gateway、VirtualService。配置Gateway中的负载loadBalancer，配置Prometheus Deployment中负载工作节点。
创建Grafana相关Gateway、VirtualService。配置Gateway中的负载loadBalancer，配置Grafana Deployment中负载工作节点。
```shell
$ kubectl apply -f prometheus.yaml
$ kubectl apply -f grafana.yaml
$ kubectl apply -f gateway-metrics.yaml
```
### **Nginx**
配置nginx相关Deployment、Service、Gateway、VirtualService。更改Deployment中负载节点、CPU、Memory限制，更改Gateway中负载loadBalancer。
> 部署前确定将其部署到istio-test命名空间，手动更改
```shell
$ kubectl apply -f nginx-istio-test.yaml
```

## **测试计划**
1. 对K8S进行测试
2. 对Istio进行测试
3. 对Istio with no Mixer进行测试

### **测试前准备**
1. 两台四核EC2实例，运行istio-ingressgateway，istio-telemetry（有待商榷）
2. 两台或者四台两核EC2实例，负载nginx服务
3. 观测工具准备
4. 观测数据：
    * 各机器CPU、内存使用情况
    * 各组件CPU、内存使用情况
    * QPS
    * 平均、90、99延迟
    * 丢包情况记录
5. 测试条件
   * 最大性能测试，记录性能使用情况
   * CPU使用率60下观测数据
   * 对每一次测试，都对前一次测试时的最大性能条件进行测试，观测该条件下性能结果，以做对比。

### **测试步骤**
1. 固定线程数，增加连接数，直到性能拐点。8、16、32、64、128、256。
2. 固定第一步连接数，增加线程数，直到性能拐点。1、2、4、8、16。
3. 限制节点CPU使用情况，重复1、2步骤。

```shell
$ wrk -t4 -c32 -d60s --latency http://<your-nginx-IP>
```











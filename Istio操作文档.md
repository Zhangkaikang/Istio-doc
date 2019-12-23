# Istio操作文档

标签（空格分隔）： istio 实习

---
[TOC]
## **安装Istio**
### **安装前提**
1.  搭建好K8S环境
2. 下载istio,将istioctl工具添加进环境变量
```
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.4.0
$ export PATH=$PWD/bin:$PATH
```  
### **安装**
#### **标准安装**
官方提供了已有安装方案如下：  
    
* default
* demo
* minimal
* sds
* remote

不同安装方案的区别，详见[官方安装方案](https://istio.io/docs/setup/additional-setup/config-profiles/)。
```
$ istioctl manifest apply --set profile=demo
```
#### **自定义安装**

istioctl manifest apply 默认default安装方案，在此基础上进行自定义安装
默认安装组件为:

* istio-citadel
* istio-galley
* istio-ingressgateway
* istio-pilot
* istio-sidecar-injector
* prometheus

```
$ istioctl manifest apply  
//禁用Mixer组件
$ istioctl manifest apply --set telemetry.enabled=false --set policy.enabled=false
//禁用Mixer组件中的policy
$ istioctl manifest apply --set telemetry.enabled=false --set policy.enabled=false
//禁用Mixer组件中的telemetry
$ istioctl manifest apply --set telemetry.enabled=false
//启用istio相关观测服务组件
$ istioctl manifest apply --set values.grafana.enabled=true --set values.kiali.enabled=true --set values.prometheus.enabled=true --set values.tracing.enabled=true --set values.gateways.enabled=true --set values.gateways.istio-ingressgateway.enabled=true --set values.gateways.istio-ingressgateway.sds.enabled=true
```
除了使用在命令中加入--set标签，还可以将定制化的功能写进yaml文件，以完成复杂的功能设置，以及便于查看。
例如禁用telemetry组件，telemetry-off.yaml
``` Yaml
apiVersion: install.istio.io/v1alpha2
kind: IstioControlPlane
spec:
  telemetry:
    enabled: false
```
```
$ istioctl manifest apply -f telemetry-off.yaml
```
>注意：每一次在使用istio manifest apply 命令时，都相当于重新安装istio，因此在安装之前应该明确需要安装的组件以及服务。
具体信息[参见官方文档](https://istio.io/docs/setup/install/istioctl/)

#### **更新配置**

istio 1.4 及以上版本支持。
```
istioctl experimental upgrade -f `<your-custom-configuration-file>`
```
详细信息查看[Upgrade istio using istioctl](https://istio.io/docs/setup/upgrade/istioctl-upgrade/)

#### **卸载Istio**
```
istioctl manifest generate --set profile=demo | kubectl delete -f -
```

### **istio-injection**
```
//为某一命名空间，设置istio-infection,之后部署在该命名空间上的Deployment，会自动在每一个Pod中加入istio-proxy容器。在此之前部署的Deployment中的Pod在重启时会自动加入istio-proxy容器
$ kubectl label namespace <namespace> istio-injection=enabled
//只为某一服务开启Istio服务
$ istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
```

## **压测方法**

1、K8S上进行Nginx压测，为之后的压测增加参考指标
2、Istio上对Nginx进行压测
3、Istio卸载Mixer之后对Nginx进行压测

### **K8S上压测**
1. 安装Nginx
```
$ cd ~/nginx-yaml
//创建deployment/nginx-app 
$ kubectl apply -f nginx-istio-test.yaml 
//将my-nginx暴露出来，自动创建相应Service/my-service
$ kubectl expose deployment my-nginx --type=LoadBalancer --name=my-service 
//重启nginx-app的Pod
$ kubectl scale deployment my-nginx --replicas=0 
$ kubectl scale deployment my-nginx --replicas=8 
//查看service、pods
$ kubectl get svc my-service -o wide 
$ kubectl get pods my-nginx -o wide
```
2、使用Wrk进行压测
```
//获取网关
$ kubectl get svc my-service -o wide
//访问上述命令得到的网址，查看是否可以访问成功
$ curl http://······
//使用wrk进行压测
$ wrk -t4 -c64 -d120s --latency http://·······
//查看结果
```
3、观测方法
由于本方案只基于K8S，只观测工作节点的CPU使用情况

### Istio上压测
1、卸载K8S上压测时使用的nginx相关Deployment、Service
```
$ kubectl get deployment 
$ kubectl delete deployment my-nginx
$ kubectl get svc
$ kubectl delete svc my-service
```
2、安装Istio和nginx
```
//将istioctl工具加进环境变量,若已加，忽略这一步
$ cd ~/istio-1.4.0
$ export PATH=$PWD/bin:$PATH
//安装istio
$ istioctl manifest apply --set profile=demo
//查看istio-system命名空间下istio组件
$ kubectl get svc -n istio-system
//安装nginx，使用的镜像为EKS工作节点自带的nginx:latest,安装的命名空间为default
$ cd ~/nginx-yaml
$ kubectl apply -f nginx-istio-test.yaml
//查看nginx相关服务
$ kubectl get deployment
$ kubectl get svc
$ kubectl get gateway
$ kubectl get virtualservice
//安装第三方观测服务grafana、prometheus、kiali、istio-tracing,对应的gateway、VirtualService,安装命名空间为istio-system
$ cd ~/istio-yaml
$ kubectl apply -f gateway-metrics.yaml
$ kubectl get svc -n istio-system
$ kubectl get gateway -n istio-system
```
3、压测
```
//获取istio-ingressgateway暴露的IP地址
$ kubectl get svc -n istio-system -o wide
$ curl http://·····
//wrk 对该IP压测
$ wrk  -t4 -c64 -d120s http://····
```
4、观测方法
```
//在第2步中，我们已经安装了相应的观测组件网关，通过访问对应的端口即可查看
//通过第3步中获取的IP地址，进行访问
//浏览器中打开grafana http://·····:15031
//主要观测页面为istio performance dashboard、istio pilot dashboard、istio mixer dashboard
//除此之外通过AWS控制台，观察各节点CPU使用情况
```

### **istio去除Mixer组件压测**
 总体过程和istio压测一致，除了安装istio时命令不同
```
 $ istioctl manifest apply --set telemetry.enabled=false --set policy.enabled=false --set values.grafana.enabled=true --set values.kiali.enabled=true --set values.prometheus.enabled=true --set values.tracing.enabled=true --set values.gateways.enabled=true --set values.gateways.istio-ingressgateway.enabled=true --set values.gateways.istio-ingressgateway.sds.enabled=true
```




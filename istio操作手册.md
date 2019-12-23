## <b>Istio操作手册</b>

### **istio安装配置**
1. 搭建K8S（EKS）环境
2. 安装istio

### **No Mixer安装**
```
使用istioctl自定义安装
istioctl manifest apply --set telemetry.enabled=false policy.enabled=false  
加入遥测组件  
istioctl manifest apply --set telemetry.enabled=false --set values.gateways.enabled=true --set values.gateways.istio-ingressgateway.enabled=true --set values.gateways.istio-ingressgateway.sds.enabled=true --set values.grafana.enabled=true --set values.kiali.enabled=true --set values.prometheus.enabled=true --set values.tracing.enabled=true

注意：每一次卸载istio后，应用程序并没有被卸载，service仍然服务在K8S上，但是由于istio一直控制着网关、流量控制，导致卸载后服务外部不可访问。下一次安装时只需要kubectl apply newGateway.yaml，找到新的hostname:port，就可以再度访问。
```
重新安装后，需要再次配置遥测软件，验证Mixer确实已经没有请求通过。  
为Grafana、Prometheus、Kiali配置外部访问。点击[官方文档 Remotely Accessing Telemetry Addons](https://istio.io/zh/docs/tasks/observability/gateways/)。

### **只部署Server Sidecar**
1. Sidecar取消自动注入，sidecar-injector=false，手动配置Deployment。  
2. 或者分配不同的命名空间，将Server Sidecar和Client Sidecar分开，Server Sidecar中采取自动注入，Client Sidecar 中取消自动注入。
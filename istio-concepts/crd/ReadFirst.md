# Istio的CRD自定义资源

## 流量管理相关

使用了Pilot相关的CRD，可以将流量与基础设施解耦，将流量管理下沉到Istio去处理，通过Pilot指定流量遵循什么规则，请求对应的目标服务。相关的资源有：
+ [VirtualService](VirtualService.md): 用于定义路由规则，并设置一些超时/重试策略
+ [DestinationRule](DestinationRule.md): 用于定义目标服务/upstream的配置策略和可路由的子集，同时可以对upstream设置对应的策略（负载均衡/断路器/tls等）
+ [ServiceEntry](ServiceEntry.md): 用于网格内对外的服务发出请求
+ [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/): 配置网关用于服务对外访问
+ [EnvoyFilter](EnvoyFilter.md): 用于定制Istio-pilot生成的代理配置，为Envoy配置过滤器，动态改变Envoy的filter chains。
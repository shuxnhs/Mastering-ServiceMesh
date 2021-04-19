# Aeraki分析



## 整体架构
![](../../static/img/istio-ecology/aeraki/architecture.png)

简单来讲，`Aeraki` 就是来帮助我们去生成[envoyfilter](../../istio-concepts/crd/EnvoyFilter.md)，即EnvoyFilter的模版生成、转换、管理插件。对于`Thrift` 和 `Dubbo` 这样的 RPC 协议（语义和 HTTP 类似），`Aeraki `沿用其原生的CRD ： [VirtualService](../../istio-concepts/crd/VirtualService.md) 和 [DestinationRule](../../istio-concepts/crd/DestinationRule.md)，并根据CRD定义的路由规则和流量规则去生成对应的EnvoyFilter；对于非 RPC 协议，`Aeraki` 则定义了一些新的 CRD 来进行管理，例如针对redis服务定义了 RedisService 和 RedisDestination并根据里面的定义去生成对应的EnvoyFilter。



## 代码设计



### 代码目录

```shell
.
├── README.md										# README文档
├── cmd
│   └── aeraki									# main函数入口
├── common-protos								# 一些pb相关的
├── demo												# aeraki提供的服务样例yaml文件
│   ├── aeraki-demo.json			  # grafana导出的配置
│   ├── dubbo										# dubbo服务的yaml，包括dubbo的deployment，destinationRule，virtualService，serviceEntry
│   ├── gateway									# kiali,prometheus,grafana的一些gateway，service
│   ├── install-demo.sh					# 安装aeraki脚本(包括istio、kiali、prometheus、grafana、dubbo、thrift、kafka等)
│   ├── kafka										# kafka相关脚本
│   ├── thrift									# thrift相关yaml
│   └── uninstall-demo.sh				# 卸载脚本
├── docker
│   └── Dockerfile							# aeraki的dockerfile
├── docs												# 文档
├── go.mod
├── go.sum
├── k8s
│   └── aeraki.yaml							# aeraki的yaml文件
├── pkg													# aeraki的核心代码
│   ├── bootstrap								# aeraki的server代码
│   ├── config									# configController 监听Istio config xDS server的配置变更
│   ├── envoyfilter							# envoyFilterController 生成对应的envoyFilter
│   ├── kube										#	与k8s的apiserve的交互
│   └── model										# 一些定义还有协议的识别（根据PortName）
├── plugin											# 各个协议插件对应的Generator实例实现
│   ├── dubbo
│   ├── kafka
│   ├── redis
│   ├── thrift
│   └── zookeeper
├── test 												# 一些yaml文件与脚本
└── vendor											# vendor
```


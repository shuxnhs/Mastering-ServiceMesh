# Aeraki分析



## 整体架构
![](../../static/img/istio-ecology/aeraki/architecture.png)

简单来讲，`Aeraki` 就是来帮助我们去生成[envoyfilter](../../istio-concepts/crd/EnvoyFilter.md)，即EnvoyFilter的模版生成、转换、管理插件。对于`Thrift` 和 `Dubbo` 这样的 RPC 协议（语义和 HTTP 类似），`Aeraki `沿用其原生的CRD ： [VirtualService](../../istio-concepts/crd/VirtualService.md) 和 [DestinationRule](../../istio-concepts/crd/DestinationRule.md)，并根据CRD定义的路由规则和流量规则去生成对应的EnvoyFilter；对于非 RPC 协议，`Aeraki` 则定义了一些新的 CRD 来进行管理，例如针对redis服务定义了 RedisService 和 RedisDestination并根据里面的定义去生成对应的EnvoyFilter。



## 代码设计



### 代码目录

```shell
.
├── README.md						# README文档
├── cmd
│   └── aeraki						# main函数入口
├── common-protos					# 一些pb相关的
├── demo							# aeraki提供的服务样例yaml文件
│   ├── aeraki-demo.json			# grafana导出的配置
│   ├── dubbo						# dubbo服务的yaml，包括dubbo的deployment，destinationRule，virtualService，serviceEntry
│   ├── gateway						# kiali,prometheus,grafana的一些gateway，service
│   ├── install-demo.sh				# 安装aeraki脚本(包括istio、kiali、prometheus、grafana、dubbo、thrift、kafka等)
│   ├── kafka						# kafka相关脚本
│   ├── thrift						# thrift相关yaml
│   └── uninstall-demo.sh			# 卸载脚本
├── docker
│   └── Dockerfile					# aeraki的dockerfile
├── docs							# 文档
├── go.mod
├── go.sum
├── k8s
│   └── aeraki.yaml					# aeraki的yaml文件
├── pkg								# aeraki的核心代码
│   ├── bootstrap					# aeraki的server代码
│   ├── config						# configController 监听Istio config xDS server的配置变更
│   ├── envoyfilter					# envoyFilterController 生成对应的envoyFilter
│   ├── kube						#	与k8s的apiserve的交互
│   └── model						# 一些定义还有协议的识别（根据PortName）
├── plugin							# 各个协议插件对应的Generator实例实现
│   ├── dubbo
│   ├── kafka
│   ├── redis
│   ├── thrift
│   └── zookeeper
├── test 							# 一些yaml文件与脚本
└── vendor						    # vendor
```



### 核心代码

#### 初始化

服务的main函数总入口，主要是去加载各个协议的Generators还有做一些日志的初始化工作，开启aeraki的server端。

```go
//------------------source: aeraki/cmd/aeraki/main.go-------------------------//
func main() {
   args := bootstrap.NewAerakiArgs()
   args.IstiodAddr = *flag.String("istiodaddr", defaultIstiodAddr, "Istiod xds server address")
   args.Namespace = *flag.String("namespace", defaultNamespace, "Current namespace")
   args.ElectionID = *flag.String("electionID", defaultElectionID, "ElectionID to elect master controller")
   args.LogLevel = *flag.String("logLevel", defaultLogLevel, "Component log level")
   flag.Parse()
   setLogLevels(args.LogLevel)
   // Create the stop channel for all of the servers.
   stopChan := make(chan struct{}, 1)
   args.Protocols = initGenerators()
   server := bootstrap.NewServer(args)
   server.Start(stopChan)

   signalChan := make(chan os.Signal, 1)
   signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)
   <-signalChan
   stopChan <- struct{}{}
}

func initGenerators() map[protocol.Instance]envoyfilter.Generator {
   return map[protocol.Instance]envoyfilter.Generator{
      protocol.Dubbo:     dubbo.NewGenerator(),
      protocol.Thrift:    thrift.NewGenerator(),
      protocol.Kafka:     kafka.NewGenerator(),
      protocol.Zookeeper: zookeeper.NewGenerator(),
   }
}
```

#### bootstrap

aeraki的server结构主要包括了本身运行的一些配置args还有就是几个controller。

```go
//------------------source: aeraki/pkg/bootstrap/server.go-----------------------//
type Server struct {
	args                  *AerakiArgs
	configController      *config.Controller				// configController监听istio配置变更
	envoyFilterController *envoyfilter.Controller			// 生成envoyfilter的controller
	crdController         manager.Manager					// k8s的crdcontroller
	stopCRDController     func()	
}

// Aeraki运行需要的参数
type AerakiArgs struct {
	IstiodAddr string		
	ListenAddr string
	Namespace  string
	ElectionID string
	LogLevel   string
	Protocols  map[protocol.Instance]envoyfilter.Generator
}


// 新建server实例，初始化各个controller
func NewServer(args *AerakiArgs) *Server {
    // configController实例化
	configController := config.NewController(args.IstiodAddr)			
    // envoyFilterController实例化
	envoyFilterController := envoyfilter.NewController(configController.Store, args.Protocols)
    // crdController实例化
	crdController := controller.NewManager(args.Namespace, args.ElectionID, func() error {
		envoyFilterController.ConfigUpdate(model.EventUpdate)
		return nil
	})

	cfg := crdController.GetConfig()
	args.Protocols[protocol.Redis] = redis.New(cfg, configController.Store)

    // configController事件处理handler，如果有配置添加/更新/删除则交给envoyFilterController去对应处理
	configController.RegisterEventHandler(args.Protocols, func(_, curr istioconfig.Config, event model.Event) {
		envoyFilterController.ConfigUpdate(event
	})

	return &Server{
		args:                  args,
		configController:      configController,
		envoyFilterController: envoyFilterController,
		crdController:         crdController,
	}
}

// Start starts all components of the Aeraki service. Serving can be canceled at any time by closing the provided stop channel.
// This method won't block
func (s *Server) Start(stop <-chan struct{}) {
	aerakiLog.Info("Staring Aeraki Server")

	go func() {
		aerakiLog.Infof("Starting Envoy Filter Controller")
		s.envoyFilterController.Run(stop)
	}()

	go func() {
		aerakiLog.Infof("Watching xDS resource changes at %s", s.args.IstiodAddr)
		s.configController.Run(stop)
	}()

	ctx, cancel := context.WithCancel(context.Background())
	s.stopCRDController = cancel
	go func() {
		_ = s.crdController.Start(ctx)
	}()

	s.waitForShutdown(stop)
}
```



#### configController

configController主要就是监听istio的配置ServiceEntry、VirtualService、DestinationRule变化，依赖了istio的[xds-mcp-api](https://github.com/istio/api/tree/master/mcp).

```go
//------------------source: aeraki/pkg/config/configcontroller.go-----------------------//
// Controller watches Istio config xDS server and notifies the listeners when config changes.
type Controller struct {
	configServerAddr string
	Store            istiomodel.ConfigStore			  // istiomodel => istio.io/istio/pilot/pkg/model
	controller       istiomodel.ConfigStoreCache	  // 监控ConfigStore
}

func (c *Controller) Run(stop <-chan struct{}) {
	go func() {
		for {
			// 拿到istio的xdsMCP
			xdsMCP, err := adsc.New(c.configServerAddr, &adsc.Config{
				Meta: istiomodel.NodeMetadata{
					Generator: "api",
				}.ToStruct(),
				InitialDiscoveryRequests: c.configInitialRequests(),
				BackoffPolicy:            backoff.NewConstantBackOff(time.Second),
			})
      
			if err != nil {
				controllerLog.Errorf("failed to dial XDS %s %v", c.configServerAddr, err)
				time.Sleep(5 * time.Second)
				continue
			}
      
			xdsMCP.Store = istiomodel.MakeIstioStore(c.controller)
			if err = xdsMCP.Run(); err != nil {
				controllerLog.Errorf("adsc: failed running %v", err)
				time.Sleep(5 * time.Second)
				continue
			}
			c.controller.Run(stop)
			return
		}
	}()
}
```

configController中的controller就是监控istio-ConfigStore的变化，有更新则会通过eventChanel通知

```go
//------------------source: istio.io/istio/pilot/pkg/config/memory/controller.go---------------//
type controller struct {
	monitor     Monitor
	configStore model.ConfigStore
}

// NewController return an implementation of model.ConfigStoreCache
// This is a client-side monitor that dispatches events as the changes are being
// made on the client.
func NewController(cs model.ConfigStore) model.ConfigStoreCache {
	out := &controller{
		configStore: cs,
		monitor:     NewMonitor(cs),
	}
	return out
}

func (c *controller) Run(stop <-chan struct{}) {
	c.monitor.Run(stop)
}
 
//------------------source: istio.io/istio/pilot/pkg/config/memory/monitor.go---------------------//
// Monitor provides methods of manipulating changes in the config store
type Monitor interface {
	Run(<-chan struct{})
	AppendEventHandler(config2.GroupVersionKind, Handler)
	ScheduleProcessEvent(ConfigEvent)
}

func NewMonitor(store model.ConfigStore) Monitor {
	return newBufferedMonitor(store, BufferSize, false)
}

func (m *configstoreMonitor) Run(stop <-chan struct{}) {
	if m.sync {
		<-stop
		return
	}
	for {
		select {
		case <-stop:
			return
		case ce, ok := <-m.eventCh:
			if ok {
                // eventChannel的event处理
				m.processConfigEvent(ce)
			}
		}
	}
}

func (m *configstoreMonitor) processConfigEvent(ce ConfigEvent) {
	if _, exists := m.handlers[ce.config.GroupVersionKind]; !exists {
		log.Warnf("Config GroupVersionKind %s does not exist in config store", ce.config.GroupVersionKind)
		return
	}
	m.applyHandlers(ce.old, ce.config, ce.event)
}
```

configController处理event的handler

```go
//------------------source: aeraki/pkg/config/configcontroller.go-----------------------//

// RegisterEventHandler adds a handler to receive config update events for a configuration type
func (c *Controller) RegisterEventHandler(protocols map[protocol.Instance]envoyfilter.Generator, 
                                          handler   func(istioconfig.Config, istioconfig.Config, istiomodel.Event)) {
  
  
	handlerWrapper := func(prev istioconfig.Config, curr istioconfig.Config, event istiomodel.Event) {
		if event == istiomodel.EventUpdate && reflect.DeepEqual(prev.Spec, curr.Spec) {
			return
		}
    
		// ServiceEntry有变动
		if curr.GroupVersionKind == collections.IstioNetworkingV1Alpha3Serviceentries.Resource().GroupVersionKind() {
			service, ok := curr.Spec.(*networking.ServiceEntry)
			if !ok {
				// This should never happen，断言为ServiceEntry失败，不可能发生
				controllerLog.Errorf("Failed in getting a virtual service: %v", curr.Name)
				return
			}
      
            // 通过portName来识别对应的协议，必须以tcp-protocol-serviceXXX命名
			for _, port := range service.Ports {
				if !strings.HasPrefix(port.Name, "tcp") {
					continue
				}
				if _, ok := protocols[protocol.GetLayer7ProtocolFromPortName(port.Name)]; ok {
					controllerLog.Infof("Matched protocol :%s %s %s", protocol.GetLayer7ProtocolFromPortName(port.Name), event.String(), curr.Name)
                    // 执行对应的handler
					handler(prev, curr, event)
				}
			}
		} else if curr.GroupVersionKind == collections.IstioNetworkingV1Alpha3Virtualservices.Resource().GroupVersionKind() {
			// VirtualService有变动
			controllerLog.Infof("Virtual Service changed: %s %s", event.String(), curr.Name)
			vs, ok := curr.Spec.(*networking.VirtualService)
			if !ok {
				// This should never happen
				controllerLog.Errorf("Failed in getting a virtual service: %v", event.String(), curr.Name)
				return
			}
            // 获取所有的serviceEntries
			serviceEntries, err := c.Store.List(collections.IstioNetworkingV1Alpha3Serviceentries.Resource().GroupVersionKind(), "")
			if err != nil {
				controllerLog.Errorf("Failed to list configs: %v", err)
				return
			}
			for _, config := range serviceEntries {
				service, ok := config.Spec.(*networking.ServiceEntry)
        
				if !ok { // should never happen
					controllerLog.Errorf("failed in getting a service entry: %s: %v", config.Labels, err)
					return
				}
				if len(service.Hosts) > 0 {
					for _, host := range service.Hosts {
                        // 与virtualService中的host匹配上了，执行对应的handler
						if host == vs.Hosts[0] {
							for _, port := range service.Ports {
								if _, ok := protocols[protocol.GetLayer7ProtocolFromPortName(port.Name)]; ok {
									handler(prev, curr, event)
								}
							}
						}
					}
				}
			}
		}
	}

	schemas := configCollection.All()
	for _, schema := range schemas {
        // 在event处理过程中注入handlerWrapper
		c.controller.RegisterEventHandler(schema.Resource().GroupVersionKind(), handlerWrapper)
	}
}
```



#### envoyFilterController





#### crdController





#### Generators






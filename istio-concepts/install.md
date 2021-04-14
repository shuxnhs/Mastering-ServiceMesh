# 安装与命令相关

Istio安装需要特定版本的`Kubernetes`,比如Istio1.9版本兼容的版本需要≥1.17,所以我们需要先准备
一套k8s环境。

> 安装k8s的网络插件的版本也不能太低。曾经掉过一个坑，istio注入后pod跨节点网络不通，原因就是集群
> 中有一个结点安装的calico版本太低，结果calico 容器中报错，“Failed to add IPIP tunnel device”;
> "Can't enable XDP acceleration, error=kernel is too old ",calico版本太老，升级到最新版解决。





## 下载

1. 安装Istio我们可以使用官方提供的二进制文件Istioctl去安装。

2. 下载

   ```shell
   # 下载最新版本
   $ curl -L https://istio.io/downloadIstio | sh -
   # 下载对应版本可以使用ISTIO_VERSION指定，特定的处理器体系可以通过TARGET_ARCH指定，比如：
   curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.6.8 TARGET_ARCH=x86_64 sh -
   ```

3. 进入Istio-1.9.1/bin，将istioctl加入环境变量

   ```shell
   export PATH=$PATH:$PWD
   ```



## 安装

1. istioctl安装有不同的安装配置，就跟我们去买电子产品一样，我们可以根据我们的预算选择高配低配的机器。同样的我们可以根据我们集群的机器资源或者不同的环境选择合适的安装配置。

   ```shell
   # 我们可以通过istioctl profile查看有的安装配置
   $ istioctl profile list
   Istio configuration profiles:
       default
       demo
       empty
       minimal
       openshift
       preview
       remote
   ```

   >不同的profiles安装的组建不同，我们可以在官网查看：https://istio.io/latest/docs/setup/additional-setup/config-profiles/，我们也可以定制自己的profile

2. 我们假设以demo配置安装

   ```shell
   $ istioctl install --set profile=demo -y
   ```

   部署成功后我们可以通过`kubectl get po -n istio-system`去查看运行起来的po

   ```shell
   kubectl get po -n istio-system
   NAME                                    READY   STATUS    RESTARTS   AGE
   istio-egressgateway-bd477794-ngdxf      1/1     Running   1          18d
   istio-ingressgateway-79df7c789f-fvqkh   1/1     Running   1          18d
   istiod-6dc55bbdd-sx8rx                  1/1     Running   1          18d
   ```

3. 其他仪表盘，最新版本的Istio（1.9）已经不包含在profile里面了，如果我们需要查看Kiali仪表盘或者其他遥测应用，比如Prometheus，Grafana，Jaeger这些需要另外安装，安装的Yaml文件放在了samples目录

   ```shell
   $ kubectl apply -f samples/addons
   # 默认部署的只能在集群内访问，开启外部访问将service修改为LoadBalancer
   $ kubectl patch service kiali --patch '{"spec":{"type":"LoadBalancer"}}' -n istio-system
   # 获取端口
   $ kubectl -n istio-system get service kiali -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'
   ```



## 注入

1. 在某个命名空间自动注入

   ```shell
   # 为对应的命名空间打上istio-injection=enabled的标签即可自动对命名空间内的po注入
   $ kubectl label namespace default istio-injection=enabled
   ```

2. 手动注入

   ```
   istioctl kube-inject -f xx.yaml | kubectl apply -f -
   ```



## 查看sidecar的Proxy

1. 查看proxy日志

   ```shell
   # 查看proxy日志
   $ kubectl logs xx-pod-name -c istio-proxy
   ```

2. 修改proxy的日志级别

   ```shell
   # 修改proxy日志level
   $ kubectl exec xx-pod-name -c istio-proxy -- curl -XPOST -s -o /dev/null http://localhost:15000/logging?level=debug
   ```

3. 查看proxy的信息
   ```shell
   # 查看某个po的envoy
   $ istioctl d envoy sample-pod-name
   http://localhost:15000
   
   # admin的相关命令
   admin commands are:
   /: Admin home page
   /certs: print certs on machine
   /clusters: upstream cluster status
   /config_dump: dump current Envoy configs (experimental)
   /contention: dump current Envoy mutex contention stats (if enabled)
   /cpuprofiler: enable/disable the CPU profiler
   /drain_listeners: drain listeners
   /healthcheck/fail: cause the server to fail health checks
   /healthcheck/ok: cause the server to pass health checks
   /heapprofiler: enable/disable the heap profiler
   /help: print out list of admin commands
   /hot_restart_version: print the hot restart compatibility version
   /init_dump: dump current Envoy init manager information (experimental)
   /listeners: print listener info
   /logging: query/change logging levels
   /memory: print current allocation/heap usage
   /quitquitquit: exit the server
   /ready: print server state, return 200 if LIVE, otherwise return 503
   /reopen_logs: reopen access logs
   /reset_counters: reset all counters to zero
   /runtime: print runtime values
   /runtime_modify: modify runtime values
   /server_info: print server version/status information
   /stats: print server stats
   /stats/prometheus: print server stats in prometheus format
   /stats/recentlookups: Show recent stat-name lookups
   /stats/recentlookups/clear: clear list of stat-name lookups and counter
   /stats/recentlookups/disable: disable recording of reset stat-name lookup names
   /stats/recentlookups/enable: enable recording of reset stat-name lookup names
   
   # 或者通过命令直接获取
   kubectl exec sample-pod-name -c istio-proxy -- pilot-agent request GET stats
   kubectl exec sample-pod-name -c istio-proxy -- pilot-agent request GET server_info
   ```
   
4. 查看熔断情况
   ```shell
   kubectl exec sample-pod-name -c istio-proxy -- pilot-agent request GET stats | grep pending | grep sample-service
   ```




## 卸载

1. 卸载资源

   ```shell
   # 删除kiali等
   $ kubectl delete -f samples/addons
   # 删除istiod，istio-egressgateway，istio-ingressgateway
   $ istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
   # 删除命名空间
   $ kubectl delete namespace istio-system
   # 去掉命名空间的自动注入标签
    kubectl label namespace default istio-injection-
   ```

   
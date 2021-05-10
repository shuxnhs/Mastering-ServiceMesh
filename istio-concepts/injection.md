# Istio注入



## 手动注入

1. 手动注入使用istioctl工具

   ```shell
   $ istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -
   ```

   

## 自动注入

1. 原理： kubernetes的webhook能力

2. 自动注入所需条件：

   + 开启了sidecarInjectorWebhook
   + 自动注入功能需要kubernetes 1.9或更高版本
   + kubernetes环境需支持MutatingAdmissionWebhook

3. 查看istio的mutatingwebhookconfiguration，通过修改配置我们可以变更自动注入匹配的label等

   ```shell
   $ kubectl get  mutatingwebhookconfiguration istio-sidecar-injector -oyaml
   ```

4. 查看istio注入全局配置

   ```shell
   $ kubectl get cm sidecar-injector -n istio-system
   ```



### 系统全局控制

1. Istio-system的命名空间下有个sidecar-injector的configmap中设置policy=disabled字段来设置是否启用自动注入

   ```yaml
   apiVersion: v1
   data:
     config: |-
       # defaultTemplates defines the default template to use for pods that do not explicitly specify a template
       defaultTemplates: [sidecar]
       policy: enabled
       // enabled启动自动注入，disabled关闭自动注入
       alwaysInjectSelector:
         []
    neverInjectSelector:
         []
         // 不要注入的配置规则数组
   ...省略其他配置
   ```
   
2. 由于注入有可以注解、标签等多种方式来实现注入，但是pod的注解要比标签选择器有更高的优先级(sidecar.istio.io/inject: "true/false")，执行的优先级是:

   `Pod Annotations > NeverInjectSelector > AlwaysInjectSelector > Default Policy`




### Namespace级别控制

1. 获取所有命名空间的注入情况
    ```shell
    $ kubectl get ns -L istio-injection
        NAME              STATUS   AGE   ISTIO-INJECTION
        default           Active   16d
        kube-system       Active   16d
        kube-public       Active   16d
        kube-node-lease   Active   16d
        istio-system      Active   15d   disabled
        dubbo             Active   15d   enabled
    ```
    
2. 为命名空间开启/关闭自动注入

    ```shell
    # 开启自动注入
    $ kubectl label namespace default/yourNamespace istio-injection=enabled
    # 关闭自动注入
    $ kubectl label namespace default istio-injection-
    ```

3. 原理：
    ![](../static/img/istio-concepts/injection.png)

### Resource级别控制

1. 在 pod 模板规范中添加 `sidecar.istio.io/inject` 的值为`true`来覆盖默认值并启用注入; `false`来覆盖默认值并禁用注入。

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ignored
     labels:
       app: ignored
   spec:
     selector:
       matchLabels:
         app: ignored
     template:
       metadata:
         labels:
           app: ignored
         annotations:
           sidecar.istio.io/inject: "true/false"
       spec:
         containers:
         - name: ignored
           image: governmentpaas/curl-ssl
           command: ["/bin/sleep","infinity"]
   ```

   


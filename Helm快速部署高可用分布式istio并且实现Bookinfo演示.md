## Helm&Istio快速构建部署

这里我们将会使用Helm的方式快速部署构建Istio微服务

Helm和翻墙的代理部署以及Docker的流量导向，我们在前面已经讲过了，这里就不再更多的赘述。

### 使用 Helm 部署 Istio 服务


关注国内外云原生领域的技术和前沿趋势，为开发者和企业提供最新的理论和实践干货。近期我们将视角放到业界炙手可热的 微服务管理工具 Istio 之上 ，将持续更新系列干货文章

Istio 项目能够为微服务架构提供流量管理机制，同时亦为其它增值功能（包括安全性、监控、路由、连接管理与策略等）创造了基础。这款软件利用久经考验的 Lyft Envoy 代理进行构建，可在无需对应用程序代码作出任何发动的前提下实现可视性与控制能力。Istio 项目是一款强大的工具，可帮助 CTO/CIO 们立足企业内部实施整体性安全、政策与合规性要求。

Istio 至今已经开发了一年多的时间。直至今天，Istio 才推出了 1.0 版本，这是一个重要的里程碑，意味着所有的核心功能现在都可以用于生产环境。与两个月前发布的 0.8 版本相比，1.0 版本只新增了一些新功能，大部分工作主要还是用于修复错误和提升性能，将许多现有的功能标记为 Beta 状态 —— 表明可用于生产环境。

这个项目的组件相对比较复杂，原有的一些选项是靠 ConfigMap 以及 istioctl 分别调整的，现在通过重新设计的 Helm Chart ，安装选项用 values.yml 或者 helm 命令行的方式来进行集中管理了。

在安装 Istio 之前要确保 Kubernetes 集群（仅支持 v1.9 及以后版本）已部署并配置好本地的 kubectl 客户端。

### 1. 下载 Istio

    $ wget https://github.com/istio/istio/releases/download/1.0.2/istio-1.0.2-linux.tar.gz

    $ tar zxf istio-1.0.2-linux.tar.gz

    $ cp istio-1.0.2/bin/istioctl /usr/local/bin/


当然我们应该跟着Istio的最新发布版或者按照自己的意思进行分布式快速部署。

### 2. 使用 Helm 部署 Istio 服务

    $ git clone https://github.com/istio/istio.git
    $ cd istio

安装包内的 Helm 目录中包含了 Istio 的 Chart，官方提供了两种方法：

用 Helm 生成 istio.yaml ，然后自行安装。
用 Tiller 直接安装。
很明显，两种方法并没有什么本质区别，这里我们采用第一种方法来部署。

    $ helm template install/kubernetes/helm/istio --name istio --namespace istio-system --set sidecarInjectorWebhook.enabled=true --set ingress.service.type=NodePort --set gateways.istio-ingressgateway.type=NodePort --set gateways.istio-egressgateway.type=NodePort --set tracing.enabled=true --set servicegraph.enabled=true --set prometheus.enabled=true --set tracing.jaeger.enabled=true --set grafana.enabled=true > istio.yaml
    
    $ kubectl create namespace istio-system
    $ kubectl create -f istio.yaml


这里说的是使用 install/kubernetes/helm/istio 目录中的 Chart 进行渲染，生成的内容保存到 ./istio.yaml 文件之中。将 sidecarInjectorWebhook.enabled 设置为 true，从而使自动注入属性生效。

部署完成后，可以检查 isotio-system namespace 中的服务是否正常运行：

    $ kubectl -n istio-system get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
    
    istio-citadel-f5779fbbb-brbxd
    istio-cleanup-secrets-jjqg5
    istio-egressgateway-6c5cc7dd86-l2c82
    istio-galley-6bf8f6f4b7-twvzl
    istio-ingressgateway-fbfdfc5c7-fg9xh
    istio-pilot-85df58955d-g5bfh
    istio-policy-74c48c8ccb-wd6h6
    istio-sidecar-injector-cf5999cf8-h9smx
    istio-statsd-prom-bridge-55965ff9c8-2hmzf
    istio-telemetry-cb49594cc-gfd84
    istio-tracing-77f9f94b98-9xvzs
    prometheus-7456f56c96-xcdh4
    servicegraph-5b8d7b4d5-lzhth


- 过去的 istio-ca 现已更名 istio-citadel 。
- istio-cleanup-secrets 是一个 job，用于清理过去的 Istio 遗留下来的 CA 部署（包括 sa、deploy 以及 svc 三个对象）。
- egressgateway 、 ingress 以及 ingressgateway ，可以看出边缘部分的变动很大，以后会另行发文。


### 3. Prometheus、Grafana、Servicegraph 和 Jaeger

等所有 Pod 启动后，可以通过 NodePort、Ingress 或者 kubectl proxy 来访问这些服务。比如可以通过 Ingress 来访问服务。

首先为 Prometheus、Grafana、Servicegraph 和 Jaeger 服务创建 Ingress：


    $ cat ingress.yaml
    
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: prometheus
      namespace: istio-system
    spec:
      rules:
      - host: prometheus.istio.io
    http:
      paths:
      - path: /
    backend:
      serviceName: prometheus
      servicePort: 9090
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: grafana
      namespace: istio-system
    spec:
      rules:
      - host: grafana.istio.io
    http:
      paths:
      - path: /
    backend:
      serviceName: grafana
      servicePort: 3000
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: servicegraph
      namespace: istio-system
    spec:
      rules:
      - host: servicegraph.istio.io
    http:
      paths:
      - path: /
    backend:
      serviceName: servicegraph
      servicePort: 8088
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: tracing
      namespace: istio-system
    spec:
      rules:
      - host: tracing.istio.io
    http:
      paths:
      - path: /
    backend:
      serviceName: tracing
      servicePort: 80


之后执行:

    $ kubectl create -f ingress.yaml


然后在你的本地电脑上添加四条 hosts ：

    $Ingree_host prometheus.istio.io
    $Ingree_host grafana.istio.io
    $Ingree_host servicegraph.istio.io
    $Ingree_host tracing.istio.io

将 $Ingree_host 替换为 Ingress Controller 运行节点的 IP。

通过 http://grafana.istio.io 访问 Grafana 服务：

通过 http://servicegraph.istio.io 访问 ServiceGraph 服务，展示服务之间调用关系图。

http://servicegraph.istio.io/force/forcegraph.html : As explored above, this is an interactive D3.js visualization.


- http://servicegraph.istio.io/dotgraph : provides a [DOT]( https://www.wikiwand.com/en/DOT_(graph_description_language ) serialization.
- http://servicegraph.istio.io/d3graph : provides a JSON serialization for D3 visualization.
- http://servicegraph.istio.io/graph : provides a generic JSON serialization.


通过 http://tracing.istio.io/ 访问 Jaeger 跟踪页面：

Note:

如果你已经部署了 Prometheus-operator ，可以不必部署 Grafana，直接将 addons/grafana/dashboards 目录下的 Dashboard 模板复制出来放到 Prometheus-operator 的 Grafana 上，然后添加 istio-system 命名空间中的 Prometheus 数据源就可以监控 Istio 了。


### 4. Mesh Expansion

Istio 还支持管理非 Kubernetes 管理的应用。此时，需要在应用所在的 VM 或者物理中部署 Istio，具体步骤请参考 Mesh Expansion

部署好后，就可以向 Istio 注册应用，如：

    # istioctl register servicename machine-ip portname:port
    $ istioctl -n onprem register mysql 1.2.3.4 3306
    $ istioctl -n onprem register svc1 1.2.3.4 http:7000


截止目前为止（参考链接）：
http://ju.outofmemory.cn/entry/364880




## 快速部署BookInfo进行访问验证应用的可用性


#### 部署Bookinfo示例程序(这里有坑)

在下载的Istio安装包的samples目录中包含了示例应用程序。

Bookinfo应用
 
部署一个样例应用，它由四个单独的微服务构成，用来演示多种 Istio 特性。这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。
Bookinfo 应用分为四个单独的微服务：

- productpage ：productpage 微服务会调用 details 和 reviews 两个微服务，用来生成页面。
- details ：这个微服务包含了书籍的信息。
- reviews ：这个微服务包含了书籍相关的评论。它还会调用 ratings 微服务。
- ratings ：ratings 微服务中包含了由书籍评价组成的评级信息。
- reviews 微服务有 3 个版本：
- v1 版本不会调用 ratings 服务。
- v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

Bookinfo 是一个异构应用，几个微服务是由不同的语言编写的。这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 reviews 服务具有多个版本。
 
要在 Istio 中运行这一应用，无需对应用自身做出任何改变。我们只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。这个过程所需的具体命令和配置方法由运行时环境决定，而部署结果较为一致，如下图所示：

所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。
 
接下来可以根据 Istio 的运行环境，按照下面的讲解完成应用的部署。
 
先验证kubernetes 集群是否启用了 自动Sidecar 注入。


Sidecar 的自动注入
 
使用 Kubernetes 的 mutating webhook admission controller，可以进行 Sidecar 的自动注入。Kubernetes 1.9 以后的版本才具备这一能力。使用这一功能之前首先要检查 kube-apiserver 的进程，是否具备 admission-control 参数，并且这个参数的值中需要包含 

MutatingAdmissionWebhook 以及 ValidatingAdmissionWebhook 两项，并且按照正确的顺序加载，这样才能启用 admissionregistration API：

    [root@centos-110 ~]# kubectl api-versions | grep admissionregistration
    admissionregistration.k8s.io/v1beta1
     
    #ps -ef | grep apiserver


这里需要特别注意一下：
如果k8s由于版本原因没有自动注入选项我们需要手动修改一些参数使得应用更好的支持。

就拿kubernetes1.9.2版本来说。


按照系统默认的路径下我们可以追踪到的路径：

/etc/kubernetes/manifests/kube-apiserver.yaml


我们需要做的配置是：


     - --admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota,MutatingAdmissionWebhook,ValidatingAdmissionWebhook


增加两个参数信息：
MutatingAdmissionWebhook,ValidatingAdmissionWebhook


否则可能出现sidecar无法自动注入的功能，最终造成微服务发布失败。


    #ps -ef | grep apiserver

确认kubernetes 集群已启用了自动Sidecar 注入。
 
部署bookinfo 服务
 
创建 book 命名空间（可选），也可以直接使用默认的 default 命名空间。

    #kubectl create ns book



给 book 命名空间设置标签：istio-injection=enabled：


    $ kubectl label namespace book istio-injection=enabled 
    $ kubectl get namespace -L istio-injection

    [root@centos-110 ~]# kubectl get ns -L istio-injection
    NAME   STATUSAGE   ISTIO-INJECTION
    book   Active1denabled
    defaultActive27d   
    istio-system   Active8ddisabled
    kube-publicActive27d   
    kube-systemActive27d   
    weave  Active21d  


如果集群使用的是自动 Sidecar 注入，只需简单的 kubectl 就能完成服务的部署。


    [root@centos-110 istio-1.0.0]# kubectl apply -n book -f samples/bookinfo/platform/kube/bookinfo.yaml
    service "details" created
    deployment.extensions "details-v1" created
    service "ratings" created
    deployment.extensions "ratings-v1" created
    service "reviews" created
    deployment.extensions "reviews-v1" created
    deployment.extensions "reviews-v2" created
    deployment.extensions "reviews-v3" created
    service "productpage" created
    deployment.extensions "productpage-v1" created
 
上面的命令会启动全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）。

如果服务部署有问题，可以使用如下命令删除已经部署的服务。

    #kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml -n book


确认所有的服务和Pods 都正常运行：

    [root@centos-110 istio-1.0.0]# kubectl get svc -n book
    NAME  TYPECLUSTER-IP   EXTERNAL-IP   PORT(S)AGE
    details   ClusterIP   10.103.243.183   <none>9080/TCP   9s
    productpage   ClusterIP   10.111.96.136<none>9080/TCP   7s
    ratings   ClusterIP   10.111.136.187   <none>9080/TCP   9s
    reviews   ClusterIP   10.97.99.117 <none>9080/TCP   8s
     
    [root@centos-110 istio-1.0.0]# kubectl get pods -n book -o wide
    NAME   READY STATUSRESTARTS   AGE   IP NODE
    details-v1-6865b9b99d-4rpfs2/2   Running   0  12m   10.244.2.140   centos-112
    productpage-v1-f8c8fb8-9zlmb   2/2   Running   0  12m   10.244.2.150   centos-112
    ratings-v1-77f657f55d-mvx9g2/2   Running   0  12m   10.244.2.135   centos-112
    reviews-v1-6b7f6db5c5-8jfq72/2   Running   0  12m   10.244.2.142   centos-112
    reviews-v2-7ff5966b99-pfhz82/2   Running   0  12m   10.244.2.146   centos-112
    reviews-v3-5df889bcff-vhk492/2   Running   0  12m   10.244.1.147   centos-111
 
验证Pod是否会自动注入Sidecar
kubectl describe pod productpage-v1-f8c8fb8-8dnjm -n book
 
被注入Sidecar的Pod 会有2个容器，多出了一个 istio-proxy 容器及其对应的存储卷。


或者通过如下命令查看pod 中是否有istio-proxy 容器：

    [root@centos-110 ~]# kubectl get pod productpage-v1-f8c8fb8-8dnjm -o jsonpath='{.spec.containers[*].name}' -n book
    productpage istio-proxy
 
输出结果显示pod 中有2个容器，分别为 productpage 和 istio-proxy。
 
可以禁用 book 命名空间的自动注入功能，然后检查新建 Pod 是不是就不带有 Sidecar 容器了。


确定入口IP和端口
 
执行以下命令以确定您的 Kubernetes 集群是否在支持外部负载均衡器的环境中运行。
 
    [root@centos-110 ~]# kubectl get svc istio-ingressgateway -n istio-system
    NAME   TYPE   CLUSTER-IPEXTERNAL-IP   PORT(S) AGE
    istio-ingressgateway   NodePort   10.106.84.2   <none>80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32329/TCP,8060:32167/TCP,15030:31095/TCP,15031:30203/TCP   8d
     
如果 EXTERNAL-IP 设置了该值，则要求您的环境具有可用于 Ingress 网关的外部负载均衡器。如果 EXTERNAL-IP 值是 <none>（或一直是 <pending> ），则说明可能您的环境不支持为 ingress 网关提供外部负载均衡器的功能。在这种情况下，您可以使用 Service 的 node port 方式访问网关。

使用 kubectl patch 更新 istio-ingressgateway 服务网关类型
由于在我的kubernetes 集群环境中，不支持外部负载均衡器。因此，使用 node port 方式访问网关。
 
    [root@centos-110 istio-1.0.0]# kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}'
    service "istio-ingressgateway" patched
 
给 Bookinfo 应用定义 ingress gateway
现在 Bookinfo 服务都已正常运行中，我们需要让kubernetes集群外部可以访问应用，如通过浏览器访问。Istio gateway 可以实现这一目的。
 
    #kubectl apply -n book -f samples/bookinfo/networking/bookinfo-gateway.yaml


输出结果：

    gateway.networking.istio.io "bookinfo-gateway" created
    virtualservice.networking.istio.io "bookinfo" created

确认gateway创建成功：

    #kubectl get gateway -n book
    NAME   AGE
    bookinfo-gateway   15h
 
也可以通过 istioctl 命令，验证 gateway和virtualservice 创建成功，具体命令如下所示：

前面，我们通过NodePort 方式来暴露istio-ingressgateway 服务，现在根据如下命令来获取 ingress ports：

    $ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
    $ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
 
获取 ingress IP 地址：

    $ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')

通过浏览器访问 ingress 服务
 
接下来就可以在浏览器的 URL 中使用 $INGRESS_HOST:$INGRESS_PORT（也就是 192.168.56.110:31380）进行访问，输入 http://192.168.56.110:31380/productpage 网址之后，显示信息如下：

就是BookInfo页面。

如果刷新几次应用的页面，就会看到页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。


理解原理
Gateway 配置资源允许外部流量进入 Istio 服务网格，并使 Istio 的流量管理和策略功能可用于边缘服务。
在前面的步骤中，我们在 Istio 服务网格中创建了一个服务，并展示了如何将服务的 HTTP 端点暴露给外部流量。



=================================================
清理 Bookinfo 示例应用
结束对 Bookinfo 示例应用的体验之后，就可以使用下面的命令来完成应用的删除和清理了。
 
在 Kubernetes 环境中完成删除
1. 删除路由规则，并终结应用的 Pod
$ samples/bookinfo/platform/kube/cleanup.sh
 
* 确认应用已经关停 - 如果前面创建了namespace - book，则下面的命令也需要添加 -n book 参数。

    $ istioctl get gateway 
    $ istioctl get virtualservices 
    $ kubectl get pods 
 
卸载 istio
使用kubectl 卸载 istio

    #kubectl delete -f install/kubernetes/istio-demo.yaml
 
手动清除额外的job 资源

    #kubectl -n istio-system delete job --all
 
删除CRD

    #kubectl delete -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system
 
参考链接：
注入 Istio sidecar

- https://preliminary.istio.io/zh/docs/setup/kubernetes/sidecar-injection/#sidecar-%E7%9A%84%E8%87%AA%E5%8A%A8%E6%B3%A8%E5%85%A5
- Install with helm
- https://istio.io/docs/setup/kubernetes/helm-install/#option-2-install-with-helm-and-tiller-via-helm-install
- 控制 Ingress 流量
- https://preliminary.istio.io/zh/docs/tasks/traffic-management/ingress/
- Bookinfo 应用
- https://preliminary.istio.io/zh/docs/examples/bookinfo/
- Istio及Bookinfo示例程序安装试用笔记
- https://zhaohuabing.com/2017/11/04/istio-install_and_example/



参考页面：
https://www.cnblogs.com/rickie/p/istio_kubernetes.html
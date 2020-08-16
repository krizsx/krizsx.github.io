---
title: Service Mesh
date: 2019-02-05T12:21:13+08:00
lastmod: 2019-02-05T12:21:13+08:00
cover: "/img/2019/02/istio_arch.svg"
draft: true
categories: ["blog"]
tags: ["service-mesh", "rpc"]
---

微服务的未来

<!--more-->

# 现状及问题

![企业架构发展历史](/img/2019/02/history.png)

- 对代码侵入性非常强
- 每种语言都需要 SDK,来完成各种基础功能(服务注册发现,监控,降级限流等)
- 升级 SDK,每个应用都需要重启

# 背景

## 应用通讯模式发展过程

### 1. 网络流控进入操作系统

![rpc_1](/img/2019/02/rpc_1.png)

在计算机网络发展的初期, 开发人员需要在自己的代码中处理服务器之间的网络连接问题, 包括流量控制, 缓存队列, 数据加密等. 在这段时间内底层网络逻辑和业务逻辑是混杂在一起.
随着技术的发展,TCP/IP 等网络标准的出现解决了流量控制等问题.尽管网络逻辑代码依然存在,但已经从应用程序里抽离出来,成为操作系统网络层的一部分, 形成了经典的网络分层模式.

### 2. 微服务架构的出现

![rpc_2](/img/2019/02/rpc_2.png)

微服务架构是更为复杂的分布式系统,它给运维带来了更多挑战, 这些挑战主要包括资源的有效管理和服务之间的治理, 如:

- 服务注册, 服务发现
- 服务容错: 断路器, 限流, 熔断, 服务降级等等
- 灰度发布方案
- 服务调用可观测性, 指标收集
- 配置管理

在微服务架构的实现中,为提升效率和降低门槛,应用开发者会基于微服务框架来实现微服务.微服务框架一定程度上为使用者屏蔽了底层网络的复杂性及分布式场景下的不确定性.通过 API/SDK 的方式提供服务注册发现、服务 RPC 通信、服务配置管理、服务负载均衡、路由限流、容错、服务监控及治理、等通用能力, 比较典型的产品有:

- 分布式 RPC 通信框架: Thrift, gRPC 等
- 服务治理特定领域的类库和解决方案: Hystrix, Zookeeper, Zipkin, Sentinel 等
- 对多种方案进行整合的微服务框架: SpringCloud、Finagle、Dubbox 等

实施微服务的成本往往会超出企业的预期(内容多, 门槛高), 花在服务治理上的时间成本甚至可能高过进行产品研发的时间. 另外上述的方案会限制可用的工具、运行时和编程语言.微服务软件库一般专注于某个平台, 这使得异构系统难以兼容, 存在重复的工作, 系统缺乏可移植性.

Docker 和 Kubernetes 技术的流行, 为 PaaS 资源的分配管理和服务的部署提供了新的解决方案, 但是微服务领域的其他服务治理问题仍然存在.

### 3. Sidecar 模式的兴起

![rpc_3](/img/2019/02/rpc_3.png)

Sidecar(也可以叫做 agent) 在原有的客户端和服务端之间加多了一个代理, 为应用程序提供的额外的功能, 如服务发现, 路由代理, 认证授权, 链路跟踪 等等.

业界使用 Sidecar 的一些先例:

- 2013 年,Airbnb 开发了 Synapse 和 Nerve,是 sidecar 的一种开源实现
- 2014 年, Netflix 发布了 Prana,它也是一个 sidecar,可以让非 JVM 应用接入他们的 NetflixOSS 生态系统

### 4. Service Mesh(服务网格)的出现

![rpc_4](/img/2019/02/rpc_4.png)

直观地看, Sidecar 到 Service Mesh 是一个规模的升级, 不过 Service Mesh 更强调的是:

- 不再将 Sidecar(代理)视为单独的组件,而是强调由这些代理连接而形成的网络
- 基础设施, 对应用程序透明

最初的 Service Mesh 始于一个网络代理,在 2016 年 1 月业界第一个开源项目 Linkerd 发布,同年 9 月 29 日的 SF Microservices 大会上,“Service Mesh”这个词汇第一次在公开场合被使用,随后 Envoy 也发布了自己的开源版本,但此时的 Service Mesh 更多停留在 Sidecar 层面,并没有清晰的 Sidecar 管理面,因此属于 Service Mesh 的第一代.此时虽然 Service Mesh 尚不成熟,但一个初具雏形的服务间通讯层已然出现

---

## Service Mesh 的定义及发展历程

### 定义

以下是 Linkerd 的 CEO Willian Morgan 给出的 Service Mesh 的定义:

> A Service Mesh is a dedicated infrastructure layer for handling service-to-service communication. It’s responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practice, the Service Mesh is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware.

服务网格（Service Mesh）是致力于解决服务间通讯的基础设施层.它负责在现代云原生应用程序的复杂服务拓扑来可靠地传递请求.实际上,Service Mesh 通常是通过一组轻量级网络代理（Sidecar proxy）,与应用程序代码部署在一起来实现,且对应用程序透明.

### 第二代 service mesh

随后 Google 联合 IBM、Lyft 发起了 Istio 项目,从架构层面明确了数据平面、控制平面,并通过集中式的控制平面概念进一步强化了 Service Mesh 的价值,再加上巨头背书的缘故,因此 Service Mesh、Istio 概念迅速火爆起来.此时已然进入到了第二代的 Service Mesh,控制平面的概念及作用被大家认可并接受,而更重要的一点是至此已经形成了一个完整意义上的 SDN(Software Defined Network)服务通讯层.此时的 Service Mesh 架构如下图

![service_mesh](/img/2019/02/service_mesh_1.png)
![service_mesh_1](/img/2019/02/service_mesh.png)

---

# Istio

![what is istio](/img/2019/02/what_is_istio.jpg)

## 简介

通过负载平衡,服务到服务身份验证,监控等,Istio 可以轻松创建已部署服务的网络,而无需更改服务代码.通过在整个环境中部署特殊的 sidecar 代理来拦截服务的 Istio 支持,该代理拦截微服务之间的所有网络通信,然后使用其控制平面功能配置和管理 Istio,其中包括：

- HTTP,gRPC,WebSocket 和 TCP 流量的自动负载平衡.

- 通过丰富的路由规则,重试,故障转移和故障注入,对流量行为进行细粒度控制.

- 可插入的策略层和配置 API,支持访问控制,速率限制.

- 集群中所有流量的自动度量标准,日志和跟踪,包括集群入口和出口.

- 通过强大的基于身份的身份验证和授权,在集群中实现安全的服务到服务通信.

Istio 旨在实现可扩展性,满足不同的部署需求.

## 架构设计

### 数据平面 (Data Plane)

- Sidecar : 默认为 Envoy

### 控制平面 (Control Plane)

- Pilot: 服务发现、流量管理
- Mixer: 访问控制、遥测
- Citadel: 终端用户认证、流量加密

![istio_arch](/img/2019/02/istio_arch.svg)

## 术语

- Envoy: 扮演 Sidecar 的功能,协调服务网格中所有服务的出入站流量,并提供服务发现、负载均衡、限流熔断等能力,还可以收集与流量相关的性能指标.

- Pilot: 负责部署在 Service Mesh 中的 Envoy 实例的生命周期管理.本质上是负责流量管理和控制,将流量和基础设施扩展解耦,这是 Istio 的核心.可以把 Pilot 看做是管理 Sidecar 的 Sidecar, 但是这个特殊的 Sidacar 并不承载任何业务流量.Pilot 让运维人员通过 Pilot 指定它们希望流量遵循什么规则,而不是哪些特定的 pod/VM 应该接收流量.有了 Pilot 这个组件,我们可以非常容易的实现 A/B 测试和金丝雀 Canary 测试.

- Mixer: Mixer 在应用程序代码和基础架构后端之间提供通用中介层.它的设计将策略决策移出应用层,用运维人员能够控制的配置取而代之.应用程序代码不再将应用程序代码与特定后端集成在一起,而是与 Mixer 进行相当简单的集成,然后 Mixer 负责与后端系统连接.Mixer 可以认为是其他后端基础设施（如数据库、监控、日志、配额等）的 Sidecar Proxy.

- Citadel: 提供强大的服务间认证和终端用户认证,使用交互 TLS,内置身份和证书管理.可以升级服务网格中的未加密流量,并为运维人员提供基于服务身份而不是网络控制来执行策略的能力.Istio 的未来版本将增加细粒度的访问控制和审计,以使用各种访问控制机制（包括基于属性和角色的访问控制以及授权钩子）来控制和监视访问服务、API 或资源的访问者.

- Galley: Galley 代表其他 Istio 控制平面组件验证用户创作的 Istio API 配置.随着时间的推移,Galley 将接管 Istio 作为顶级配置摄取,处理和分发组件的责任.它将负责将其余的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节隔离开来.

- Routing: Kubernetes 中的 service 是没有任何路由属性可以配置的,Istio 在设计之初就通过在同一个 Pod 中,在应用容器旁运行一个 sidecar proxy 来透明得实现细粒度的路由控制.

### Istio 对 Pod 和服务的要求

要成为服务网格的一部分,Kubernetes 集群中的 Pod 和服务必须满足以下几个要求：

- 需要给端口正确命名：服务端口必须进行命名.端口名称只允许是<协议>[-<后缀>-]模式,其中<协议>部分可选择范围包括 http、http2、grpc、mongo 以及 redis,Istio 可以通过对这些协议的支持来提供路由能力.例如 name: http2-foo 和 name: http 都是有效的端口名,但 name: http2foo 就是无效的.如果没有给端口进行命名,或者命名没有使用指定前缀,那么这一端口的流量就会被视为普通 TCP 流量（除非显式的用 Protocol: UDP 声明该端口是 UDP 端口）.

- 关联服务：Pod 必须关联到 Kubernetes 服务,如果一个 Pod 属于多个服务,这些服务不能再同一端口上使用不同协议,例如 HTTP 和 TCP.

- Deployment 应带有 app 以及 version 标签：在使用 Kubernetes Deployment 进行 Pod 部署的时候,建议显式的为 Deployment 加上 app 以及 version 标签.每个 Deployment 都应该有一个有意义的 app 标签和一个用于标识 Deployment 版本的 version 标签.app 标签在分布式跟踪的过程中会被用来加入上下文信息.Istio 还会用 app 和 version 标签来给遥测指标数据加入上下文信息.

# 实验

[protolbuf 语法](https://developers.google.com/protocol-buffers/docs/proto3)

## 生产环境安装

[官网](https://istio.io/docs/setup/kubernetes/helm-install/)

参考 https://github.com/helm/helm/issues/3130

### 安装 helm 以及 tiller

```shell
helm init
# 国内被墙,需要自己拉取镜像,推到自己的镜像仓库,修改镜像源
kubectl edit deploy -n kube-system tiller-deploy

kubectl apply -f install/kubernetes/helm/helm-service-account.yaml

helm init --service-account tiller
```

![tiller](/img/2019/02/tiller.png)

### 配置 kiali

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
type: Opaque
data:
  username: YWRtaW4= # admin
  passphrase: YWRtaW4= # admin
```

必须 base64 加密,之后

```shell
kubectl apply -f ./kiali.yaml
```

### 安装所有组件

```shell
# 打开所有的组件
helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set grafana.enabled=true,kiali.enabled=true,tracing.enabled=true
```

![istio-installed](/img/2019/02/istio-installed.png)

### 卸载所有组件

```shell
# 卸载 这两步都需要
helm delete --purge istio

kubectl delete -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system

# 卸载 kiali
kubectl delete all,secrets,sa,configmaps,deployments,ingresses,clusterroles,clusterrolebindings,virtualservices,destinationrules,customresourcedefinitions --selector=app=kiali -n istio-system
```

## bookinfo demo

[官网](https://istio.io/zh/docs/examples/bookinfo/)

Bookinfo 是一个多语言异构的微服务 demo, 其中`productpage`服务会调用`details`和`reviews`两个微服务,`reviews`会调用`ratings`服务,`reviews`服务有 3 个版本

### 1.打开自动注入 sidecar

```shell
# 给default这个namespace增加label
kubectl label namespace default istio-injection=enabled
```

### 2.启动

```shell
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

![bookinfo-init](/img/2019/02/bookinfo-init.png)

启动成功之后

![bookinfo-install](/img/2019/02/bookinfo-installed.png)

### 3.定义 ingress gateway

现在 Bookinfo 服务已启动并运行，需要从 Kubernetes 集群外部访问应用程序，例如，从浏览器访问。 Istio Gateway 用于此目的。

```shell
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"
  gateways:
    - bookinfo-gateway
  http:
    - match:
        - uri:
            exact: /productpage
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

使用`kubectl get gateway`检查是否创建完成

配置 ingress 以及 gateway 的地址端口

```shell
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

```shell
$ echo $GATEWAY_URL
10.10.80.134:80
```

打开浏览器访问 http://10.10.80.134/productpage

刷新几次,可以看到有三种不同的`Book Reviews`

### 4.基于权重的路由

创建三个版本的`destinationrule`,改为随机访问

```shell
kubectl apply -f samples/bookinfo/networking/destination-rule-reviews.yaml
```

通过`VirtualService`调整个 reviews 服务子版本的流量比例, 设置 v1 和 v3 各占 50%

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
```

```shell
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
# 测试完删除
kubectl delete -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

### 5. 基于内容的路由

修改 reviews CRD, 将 jason 登录的用户版本路由到 v2, 其他用户路由到版本 v3.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v3
```

```shell
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml
# 测试完删除
kubectl delete -f samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml
```

### 6. 故障注入

为了测试我们的微服务应用程序 Bookinfo 的弹性，我们将在`reviews:v2`和`ratings`服务之间的一个用户 “jason” 注入一个 7 秒 的延迟

刷新页面, 使用 jason 登录的用户, 将看到 v2 黑色星星版本, 其他用户将看到 v1 .

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      fault:
        delay:
          percent: 100
          fixedDelay: 7s
      route:
        - destination:
            host: ratings
            subset: v1
    - route:
        - destination:
            host: ratings
            subset: v1
```

```shell
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
# 测试完删除
kubectl delete -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

### 7. 熔断

预备条件,启动一个 httpbin 服务

```
kubectl apply -f samples/httpbin/httpbin.yaml
```

创建一个熔断规则

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

现在我们已经设置了调用 httpbin 服务的规则，接下来创建一个客户端，用来向后端服务发送请求，观察是否会触发熔断策略。这里要使用一个简单的负载测试客户端，名字叫 fortio。这个客户端可以控制连接数量、并发数以及发送 HTTP 请求的延迟。这里我们会把给客户端也进行 Sidecar 的注入，以此保证 Istio 对网络交互的控制：

```shell
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
```

接下来就可以登入客户端 Pod 并使用 Fortio 工具来调用 httpbin。-curl 参数表明只调用一次：

```shell
export FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -curl  http://httpbin:8000/get
```

![fortio](/img/2019/02/fortio.png)

在上面的熔断设置中指定了 maxConnections: 1 以及 http1MaxPendingRequests: 1。这意味着如果超过了一个连接同时发起请求，Istio 就会熔断，阻止后续的请求或连接。接下来尝试一下两个并发连接（-c 2），发送 20 请求（-n 20）：

```shell
kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
```

![fortio-cb](/img/2019/02/fortio-cb.png)

这里可以看到，所有请求都通过了
Istio-proxy 允许一些余地。接下来把并发连接数量提高到 3：

![fortio-3](/img/2019/02/fortio-3.png)

测试完毕,清除规则

```shell
kubectl delete destinationrule httpbin
kubectl delete deploy httpbin fortio-deploy
kubectl delete svc httpbin
```

### 8. 卸载停止

```shell
samples/bookinfo/platform/kube/cleanup.sh
```

---

# Istio Data Plane 数据平面

## 数据平面组件

istio 注入 sidecar 实现:

- 自动注入: 利用[Kubernetes Dynamic Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)对新建的 pod 进行注入
- 手动注入: 使用`istioctl kube-inject`

注入的内容:

- istio-init: 通过配置 iptables 来劫持 pod 的流量
- istio-proxy: 会启动两个进程`pilot-agent`和`envoy`, `pilot-agent`进行初始化并启动`envoy`

### istio-init

![istio-init](/img/2019/02/istio-init.png)

```shell
$ istio-iptables.sh -p PORT -u UID -g GID [-m mode] [-b ports] [-d ports] [-i CIDR] [-x CIDR] [-h]
  -p: 指定重定向所有 TCP 流量的 Envoy 端口（默认为 $ENVOY_PORT = 15001）
  -u: 指定未应用重定向的用户的 UID。通常，这是代理容器的 UID（默认为 $ENVOY_USER 的 uid，istio_proxy 的 uid 或 1337）
  -g: 指定未应用重定向的用户的 GID。（与 -u param 相同的默认值）
  -m: 指定入站连接重定向到 Envoy 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 Envoy 中排除（可选）的入站端口列表，以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -i: 指定重定向到 Envoy（可选）的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。

环境变量位于 $ISTIO_SIDECAR_CONFIG（默认在：/var/lib/istio/envoy/sidecar.env）
```

istio-init 通过配置 iptable 来劫持 Pod 中的流量:

- 参数-p 15001: Pod 中的数据流量被 iptable 拦截，并发向 15001 端口, 该端口将由 envoy 监听
- 参数-u 1337: 用于排除用户 ID 为 1337，即 Envoy 自身的流量，以避免 Iptable 把 Envoy 发出的数据又重定向到 Envoy, UID 为 1337，即 Envoy 所处的用户空间，这也是 istio-proxy 容器默认使用的用户, 见 Sidecar istio-proxy 配置参数 securityContext.runAsUser
- 参数-b 9080 -d "": 入站端口控制, 将所有访问 9080 端口（即应用容器的端口）的流量重定向到 Envoy 代理
- 参数-i '\*' -x "": 出站 IP 控制, 将所有出站流量都重定向到 Envoy 代理

Init 容器初始化完毕后就会自动终止，但是 Init 容器初始化结果(iptables)会保留到应用容器和 Sidecar 容器中.

### istio-proxy

istio-proxy 以 sidecar 的形式注入到应用容器所在的 pod 中, 简化的注入 yaml:

```yaml
- image: docker.io/istio/proxyv2:1.0.5
  name: istio-proxy
  args:
  - proxy
  - sidecar
  - --configPath
  - /etc/istio/proxy
  - --binaryPath
  - /usr/local/bin/envoy
  - --serviceCluster
  - ratings
  - --drainDuration
  - 45s
  - --parentShutdownDuration
  - 1m0s
  - --discoveryAddress
  - istio-pilot.istio-system:15007
  - --discoveryRefreshDelay
  - 1s
  - --zipkinAddress
  - zipkin.istio-system:9411
  - --connectTimeout
  - 10s
  - --proxyAdminPort
  - "15000"
  - --controlPlaneAuthPolicy
  - NONE
  env:
    ......
  ports:
  - containerPort: 15090
    name: http-envoy-prom
    protocol: TCP
  securityContext:
    runAsUser: 1337
    ......
```

![istio-proxy](/img/2019/02/istio-proxy.png)

可以看到:

- /usr/local/bin/pilot-agent 是 /usr/local/bin/envoy 的父进程, Pilot-agent 进程根据启动参数和 K8S API Server 中的配置信息生成 Envoy 的初始配置文件(/etc/istio/proxy/envoy-rev0.json)，并负责启动 Envoy 进程
- pilot-agent 的启动参数里包括: discoveryAddress(pilot 服务地址), Envoy 二进制文件的位置, 服务集群名, 监控指标上报地址, Envoy 的管理端口, 热重启时间等

Envoy 配置初始化流程:

- Pilot-agent 根据启动参数和 K8S API Server 中的配置信息生成 Envoy 的初始配置文件 envoy-rev0.json，该文件告诉 Envoy 从 xDS server 中获取动态配置信息，并配置了 xDS server 的地址信息，即控制平面的 Pilot
- Pilot-agent 使用 envoy-rev0.json 启动 Envoy 进程
- Envoy 根据初始配置获得 Pilot 地址，采用 xDS 接口从 Pilot 获取到 Listener，Cluster，Route 等动态配置信息
- Envoy 根据获取到的动态配置启动 Listener，并根据 Listener 的配置，结合 Route 和 Cluster 对拦截到的流量进行处理

使用命令查看 envoy 初始配置

```shell
kubectl exec productpage-v1-8d69b45c-kw8ch -c istio-proxy -- cat /etc/istio/proxy/envoy-rev0.json
```

```json
{
  "node": {
    "id": "sidecar~10.31.1.152~productpage-v1-8d69b45c-kw8ch.default~default.svc.cluster.local",
    "cluster": "productpage",

    "metadata": {
      "INTERCEPTION_MODE": "REDIRECT",
      "ISTIO_PROXY_SHA": "istio-proxy:930841ca88b15365737acb7eddeea6733d4f98b9",
      "ISTIO_PROXY_VERSION": "1.0.2",
      "ISTIO_VERSION": "1.0.6",
      "POD_NAME": "productpage-v1-8d69b45c-kw8ch",
      "app": "productpage",
      "istio": "sidecar",
      "pod-template-hash": "8d69b45c",
      "version": "v1"
    }
  },
  "stats_config": {
    "use_all_default_tags": false,
    "stats_tags": [
      {
        "tag_name": "cluster_name",
        "regex": "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
      },
      {
        "tag_name": "tcp_prefix",
        "regex": "^tcp\\.((.*?)\\.)\\w+?$"
      },
      {
        "tag_name": "response_code",
        "regex": "_rq(_(\\d{3}))$"
      },
      {
        "tag_name": "response_code_class",
        "regex": "_rq(_(\\dxx))$"
      },
      {
        "tag_name": "http_conn_manager_listener_prefix",
        "regex": "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      },
      {
        "tag_name": "http_conn_manager_prefix",
        "regex": "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      },
      {
        "tag_name": "listener_address",
        "regex": "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      }
    ]
  },
  "admin": {
    "access_log_path": "/dev/null",
    "address": {
      "socket_address": {
        "address": "127.0.0.1",
        "port_value": 15000
      }
    }
  },
  "dynamic_resources": {
    "lds_config": {
      "ads": {}
    },
    "cds_config": {
      "ads": {}
    },
    "ads_config": {
      "api_type": "GRPC",
      "refresh_delay": "1s",
      "grpc_services": [
        {
          "envoy_grpc": {
            "cluster_name": "xds-grpc"
          }
        }
      ]
    }
  },
  "static_resources": {
    "clusters": [
      {
        "name": "prometheus_stats",
        "type": "STATIC",
        "connect_timeout": "0.250s",
        "lb_policy": "ROUND_ROBIN",
        "hosts": [
          {
            "socket_address": {
              "protocol": "TCP",
              "address": "127.0.0.1",
              "port_value": 15000
            }
          }
        ]
      },
      {
        "name": "xds-grpc",
        "type": "STRICT_DNS",
        "connect_timeout": "10s",
        "lb_policy": "ROUND_ROBIN",

        "hosts": [
          {
            "socket_address": {
              "address": "istio-pilot.istio-system",
              "port_value": 15010
            }
          }
        ],
        "circuit_breakers": {
          "thresholds": [
            {
              "priority": "DEFAULT",
              "max_connections": 100000,
              "max_pending_requests": 100000,
              "max_requests": 100000
            },
            {
              "priority": "HIGH",
              "max_connections": 100000,
              "max_pending_requests": 100000,
              "max_requests": 100000
            }
          ]
        },
        "upstream_connection_options": {
          "tcp_keepalive": {
            "keepalive_time": 300
          }
        },
        "http2_protocol_options": {}
      },

      {
        "name": "zipkin",
        "type": "STRICT_DNS",
        "connect_timeout": "1s",
        "lb_policy": "ROUND_ROBIN",
        "hosts": [
          {
            "socket_address": {
              "address": "zipkin.istio-system",
              "port_value": 9411
            }
          }
        ]
      }
    ],
    "listeners": [
      {
        "address": {
          "socket_address": {
            "protocol": "TCP",
            "address": "0.0.0.0",
            "port_value": 15090
          }
        },
        "filter_chains": [
          {
            "filters": [
              {
                "name": "envoy.http_connection_manager",
                "config": {
                  "codec_type": "AUTO",
                  "stat_prefix": "stats",
                  "route_config": {
                    "virtual_hosts": [
                      {
                        "name": "backend",
                        "domains": ["*"],
                        "routes": [
                          {
                            "match": {
                              "prefix": "/stats/prometheus"
                            },
                            "route": {
                              "cluster": "prometheus_stats"
                            }
                          }
                        ]
                      }
                    ]
                  },
                  "http_filters": {
                    "name": "envoy.router"
                  }
                }
              }
            ]
          }
        ]
      }
    ]
  },

  "tracing": {
    "http": {
      "name": "envoy.zipkin",
      "config": {
        "collector_cluster": "zipkin"
      }
    }
  }
}
```

## Envoy

官网(https://www.envoyproxy.io/)

[参考文章](https://jimmysong.io/posts/envoy-archiecture-and-terminology/)

### 简介

Envoy 是 Service Mesh 中一个非常优秀的 sidecar 的开源实现.包含了非常多的功能,比如:

- 动态服务发现
- 负载均衡
- TLS
- HTTP/2 和 gRPC 代理
- 断路器
- 健康检查
- 基于％的流量分割的分阶段部署
- 故障注入
- 丰富的指标

### 架构图

![envoy_arch](/img/2019/02/envoy_arch.jpg)

### 术语

- Host：能够进行网络通信的实体（在手机或服务器等上的应用程序）.在 Envoy 中主机是指逻辑网络应用程序.只要每台主机都可以独立寻址,一块物理硬件上就运行多个主机.

- Downstream：下游（downstream）主机连接到 Envoy,发送请求并或获得响应.

- Upstream：上游（upstream）主机获取来自 Envoy 的链接请求和响应.

- Cluster: 集群（cluster）是 Envoy 连接到的一组逻辑上相似的上游主机.Envoy 通过服务发现发现集群中的成员.Envoy 可以通过主动运行状况检查来确定集群成员的健康状况.Envoy 如何将请求路由到集群成员由负载均衡策略确定.

- Mesh：一组互相协调以提供一致网络拓扑的主机.Envoy mesh 是指一组 Envoy 代理,它们构成了由多种不同服务和应用程序平台组成的分布式系统的消息传递基础.

- 运行时配置：与 Envoy 一起部署的带外实时配置系统.可以在无需重启 Envoy 或 更改 Envoy 主配置的情况下,通过更改设置来影响操作.

- Listener: 侦听器（listener）是可以由下游客户端连接的命名网络位置（例如,端口、unix 域套接字等）.Envoy 公开一个或多个下游主机连接的侦听器.一般是每台主机运行一个 Envoy,使用单进程运行,但是每个进程中可以启动任意数量的 Listener（监听器）,目前只监听 TCP,每个监听器都独立配置一定数量的（L3/L4）网络过滤器.Listenter 也可以通过 Listener Discovery Service（LDS）动态获取.

- Listener filter：Listener 使用 listener filter（监听器过滤器）来操作链接的元数据.它的作用是在不更改 Envoy 的核心功能的情况下添加更多的集成功能.Listener filter 的 API 相对简单,因为这些过滤器最终是在新接受的套接字上运行.在链中可以互相衔接以支持更复杂的场景,例如调用速率限制.Envoy 已经包含了多个监听器过滤器.

- Http Route Table：HTTP 的路由规则,例如请求的域名,Path 符合什么规则,转发给哪个 Cluster.

- Health checking：健康检查会与 SDS 服务发现配合使用.但是,即使使用其他服务发现方式,也有相应需要进行主动健康检查的情况.详见 health checking.

### xDS

Envoy xDS 为 Istio 控制平面与控制平面通信的基本协议,只要代理支持该协议表达形式就可以创建自己 Sidecar 来替换 Envoy

Envoy 通过查询文件或管理服务器来动态发现资源.概括地讲,对应的发现服务及其相应的 API 被称作 xDS.Envoy 通过订阅（subscription）方式来获取资源,如监控指定路径下的文件、启动 gRPC 流或轮询 REST-JSON URL.后两种方式会发送 `DiscoveryRequest` 请求消息,发现的对应资源则包含在响应消息 `DiscoveryResponse` 中

xDS 是一个关键概念,它是一类发现服务的统称,其包括如下几类：

- CDS：Cluster Discovery Service
- EDS：Endpoint Discovery Service
- LDS：Listener Discovery Service
- RDS：Route Discovery Service
- HDS: Health Discovery Service
- ADS: Aggregated Discovery Service
- KDS: Key Discovery Service

等等

正是通过对 xDS 的请求来动态更新 Envoy 配置,概念以及 api 非常多,参考官网

![envoy_xDS](/img/2019/02/envoy_xDS.jpg)

使用这个来查看 envoy 的管理接口

```shell
kubectl exec productpage-v1-8d69b45c-kw8ch -c istio-proxy curl http://127.0.0.1:15000/help
```

![envoy-help](/img/2019/02/envoy-help.png)

## 分布式追踪

使用的是 jaeger

Envoy 原生支持 http 链路跟踪:

- 生成 Request ID：Envoy 会在需要的时候生成 UUID，并操作名为 [x-request-id] 的 HTTP Header。应用可以转发这个 Header 用于统一的记录和跟踪.

- 客户端跟踪 ID 连接：x-client-trace-id Header 可以用来把不信任的请求 ID 连接到受信的 x-request-id Header 上

跟踪上下文信息的传播

- 不管使用的是哪个跟踪服务，都应该传播 x-request-id，这样在被调用服务中启动相关性的记录
- 上下文跟踪并非零修改, 在调用下游服务时, 上游应用应该自行传播跟踪相关的 HTTP Header

# Istio Control Plane 控制平面

## Pilot

### 架构图

![pilot-adapter](/img/2019/02/pilot-adapters.svg)

- Rules API: 对外封装统一的 API，供服务的开发者或者运维人员调用，可以用于流量控制。
- Envoy API: 对内封装统一的 API，供 Envoy 调用以获取注册信息、流量控制信息等。
- 抽象模型层: 对服务的注册信息、流量控制规则等进行抽象，使其描述与平台无关。
- 平台适配层: 用于适配各个平台如 Kubernetes、Mesos、Cloud Foundry 等，把平台特定的注册信息、资源信息等转换成抽象模型层定义的平台无关的描述。例如，Pilot 中的 Kubernetes 适配器实现必要的控制器来 watch Kubernetes API server 中 pod 注册信息、ingress 资源以及用于存储流量管理规则的第三方资源的更改

### 流量管理模型

#### VirtualService

VirtualService 中定义了一系列针对指定服务的流量路由规则。每个路由规则都是针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务, 或者目标服务的子集或版本, 匹配规则中还包含了对流量发起方的定义，这样一来，规则还可以针对特定客户上下文进行定制.对于 A/B 测试和灰度发布等场景,通常需要使用划分 subset,VirtualService 中根据 destination 中的 subset 配置来选择路由,但是这些 subset 究竟对应哪些服务示例,这就需要 DestionationRule

#### DestinationRule

DestinationRule 是 `VirtualService` 路由生效后,配置应用与请求的策略集. 所定义的策略,决定了经过路由处理之后的流量的访问策略.这些策略中可以定义负载均衡配置、连接池尺寸以及外部检测（用于在负载均衡池中对不健康主机进行识别和驱逐）配置

#### Gateway

Gateway 描述了一个负载均衡器，用于承载网格边缘的进入和发出连接。这一规范中描述了一系列开放端口，以及这些端口所使用的协议、负载均衡的 SNI 配置等内容

- Ingress 和 Egress: Istio 假定进入和离开服务网络的所有流量都会通过 Envoy 代理进行传输.通过将 Envoy 代理部署在服务之前,运维人员可以针对面向用户的服务进行 A/B 测试,部署金丝雀服务等.类似地,通过使用 Envoy 将流量路由到外部 Web 服务（例如,访问 Maps API 或视频服务 API）的方式,运维人员可以为这些服务添加超时控制、重试、断路器等功能,同时还能从服务连接中获取各种细节指标

![istio-request-flow](/img/2019/02/istio_request_flow.svg)

#### ServiceEntry

Istio 服务网格内部会维护一个与平台无关的使用通用模型表示的服务注册表，当你的服务网格需要访问外部服务的时候，就需要使用 ServiceEntry 来添加服务注册, 这类服务可能是网格外的 API，或者是处于网格内部但却不存在于平台的服务注册表中的条目（例如需要和 Kubernetes 服务沟通的一组虚拟机服务）.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  ports:
    - number: 80
      name: http
      protocol: HTTP
  resolution: DNS

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
    - timeout: 3s
      route:
        - destination:
            host: httpbin.org
          weight: 100
```

#### EnvoyFilter

EnvoyFilter 对象描述了针对 sidecar 的过滤器,这些过滤器可以定制由 Istio Pilot 生成的代理配置.这一功能一定要谨慎使用.错误的配置内容一旦完成传播,可能会令整个服务网格进入瘫痪状态.详情请参考 EnvoyFilter.Envoy 中的 listener 可以配置多个 filter,这也是一种通过 Istio 来扩展 Envoy 的机制.

### Kubernetes, Istio, Envoy xDS 模型对比

|  kubernetes  |  istio   |    Envoy xDS    |
| :----------: | :------: | :-------------: |
|   入口流量   | Ingress  |     Gateway     | Listener |
|   服务定义   | Service  |        -        | Cluster+Listener |
| 外部服务定义 |    -     |  ServiceEntry   | Cluster+Listener |
|   版本定义   |    -     | DestinationRule | Cluster+Listener |
|   版本路由   |    -     | VirtualService  | Route |
|     实例     | Endpoint |        -        | Endpoint |

## Mixer

### 架构图

![mixer-arch](/img/2019/02/mixer-arch.svg)

Mixer 提供三个核心功能：

- 前提条件检查。在响应来自服务调用者的传入请求之前，使得调用者能够验证许多前提条件。允许服务在响应来自服务消费者的传入请求之前验证一些前提条件。前提条件可以包括服务使用者是否被正确认证，是否在服务的白名单上，是否通过 ACL 检查等等。
- 配额管理。 使服务能够在多个维度上分配和释放配额，当服务消费者对有限的资源发生抢占时，配额就成了一种有效的资源管理工具，它为服务之间的资源抢占提供相对的公平。频率控制就是配额的一个实例。
- 遥测报告。使服务能够上报日志和监控。在未来，它还将启用针对服务运营商以及服务消费者的跟踪和计费流。

也就是说，为了让 mixer 能够工作，就需要 envoy 从每次请求中获取信息，然后发起两次对 mixer 的请求：

- 在转发请求之前：这时需要做前提条件检查和配额管理，只有满足条件的请求才会做转发
- 在转发请求之后：这时要上报日志等，术语上称为遥感信息，Telemetry，或者 Reporting。

### Mixer Adapter 模型

#### Attribute

Attribute 是策略和遥测功能中有关请求和环境的基本数据, 是用于描述特定服务请求或请求环境的属性的一小段数据。例如，属性可以指定特定请求的大小、操作的响应代码、请求来自的 IP 地址等.

- Istio 中的主要属性生产者是 Envoy，但专用的 Mixer 适配器也可以生成属性
- 属性词汇表见: [Attribute Vocabulary](https://istio.io/docs/reference/config/policy-and-telemetry/attribute-vocabulary/)
- 数据流向: envoy -> mixer

#### Template

Template 是对 adapter 的数据格式和处理接口的抽象, Template 定义了:

- 当处理请求时发送给 adapter 的数据格式
- adapter 必须实现的 gRPC service 接口

每个 Template 通过 template.proto 进行定义:

- 名为 Template 的一个 message
- Name: 通过 template 所在的 package name 自动生成
- template_variety: 可选 Check, Report, Quota or AttributeGenerator, 决定了 adapter 必须实现的方法. 同时决定了在 mixer 的什么阶段要生成 template 对应的 instance:
  - Check: 在 Mixer’s Check API call 时创建并发送 instance
  - Report: 在 Mixer’s Report API call 时创建并发送 instance
  - Quota: 在 Mixer’s Check API call 时创建并发送 instance(查询配额时)
  - AttributeGenerator: for both Check, Report Mixer API calls

#### Instance

代表符合某个 Template 定义的数据格式的具体实现, 该具体实现由用户配置的 CRD, CRD 定义了将 Attributes 转换为具体 instance 的规则, 支持属性表达式

- Instance CRD 是 Template 中定义的数据格式 + 属性转换器
- 内置的 Instance 类型(其实就是内置 Template): Templates
- 属性表达式见: Expression Language
- 数据流向: mixer -> adapter 实例

#### Adapter

封装了 Mixer 和特定外部基础设施后端进行交互的必要接口，例如 Prometheus 或者 Stackdriver

- 定义了需要处理的模板(在 yaml 中配置 template)
- 定义了处理某个 Template 数据格式的 GRPC 接口
- 定义 Adapter 需要的配置格式(Params)
- 可以同时处理多个数据(instance)

#### Handler

用户配置的 CRD, 为具体 Adapter 提供一个具体配置, 对应 Adapter 的可运行实例

#### Rule

用户配置的 CRD, 配置一组规则，这些规则描述了何时调用特定(通过 Handler 对应的)适配器及哪些 Instance

# 总结

![mind-node](/img/2019/02/istio-mind-node.png)

# 迁移到 Service Mesh 需要的工作

- 容器化服务
- 切换 gRPC 协议
- istio 配置管理
- 性能测试

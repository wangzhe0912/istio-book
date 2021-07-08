# Istio 流量管理

Istio 的流量路由规则可以让我们很容易的控制服务之间的流量和 API 调用。

Istio 简化了服务级别属性的配置，比如熔断器、超时和重试，并且能轻松的设置重要的任务，如 A/B 测试、金丝雀发布、基于流量百分比切分的概率发布等。

它还提供了开箱即用的故障恢复特性，有助于增强应用的健壮性，从而更好地应对被依赖的服务或网络发生故障的情况。

Istio 的流量管理模型源于和服务一起部署的 Envoy 代理。

网格内服务发送和接收的所有流量（data plane流量）都经由 Envoy 代理，这让控制网格内的流量变得异常简单，而且不需要对服务做任何的更改。

本节中描述的功能特性，如果我们对它们是如何工作的感兴趣的话，可以在 [架构概述](../chap05/architecture.md) 中找到关于 Istio 的流量管理实现的更多信息。

本部分只介绍 Istio 的流量管理特性。

## 概述

为了在服务网格中导流，Istio 需要知道所有的 endpoint 在哪和它属于哪个服务。

为了定位到service registry(服务注册中心)，Istio 会连接到一个服务发现系统。

例如，如果我们在 Kubernetes 集群上安装了 Istio，那么它将自动检测该集群中的服务和 endpoint。

使用此服务注册中心，Envoy 代理可以将流量定向到相关服务。

大多数基于微服务的应用程序，每个服务的工作负载都有多个实例来处理流量，称为负载均衡池。
默认情况下，Envoy 代理基于轮询调度模型在服务的负载均衡池内分发流量，按顺序将请求发送给池中每个成员，一旦所有服务实例均接收过一次请求后，重新回到第一个池成员。

Istio 基本的服务发现和负载均衡能力为我们提供了一个可用的服务网格，但它能做到的远比这多的多。

在许多情况下，我们可能希望对网格的流量情况进行更细粒度的控制。

作为 A/B 测试的一部分，我们可能想将特定百分比的流量定向到新版本的服务，或者为特定的服务实例子集应用不同的负载均衡策略。

我们可能还想对进出网格的流量应用特殊的规则，或者将网格的外部依赖项添加到服务注册中心。

通过使用 Istio 的流量管理 API 将流量配置添加到 Istio，就可以完成所有这些甚至更多的工作。

和其他 Istio 配置一样，这些 API 也使用 Kubernetes 的自定义资源定义（CRDs）来声明，我们可以像示例中看到的那样使用 YAML 进行配置。

接下来，我们将会对 Istio 流量管理中用到的这些 API 资源对象依次进行讲解和说明。

## VirtualService

VirtualService 和 DestinationRule 是 Istio 流量路由功能中最核心的对象。

VirtualService 让我们配置如何在服务网格内将请求路由到服务，这基于 Istio 和平台提供的基本的连通性和服务发现能力。

每个 VirtualService 包含一组路由规则，Istio 按顺序去评估它们，Istio 将每个给定的请求匹配到VirtualService指定的实际目标地址。

### 为什么需要 VirtualService

VirtualService 在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用。

通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。

VirtualService同时提供了丰富的方式，为发送至这些工作负载的流量指定不同的路由规则。

为什么 VirtualService 如此有用？

就像在之前的介绍中所说，如果没有VirtualService，Envoy 会在所有的服务实例中使用轮询的负载均衡策略分发请求。 我们可以用我们对工作负载的了解来改善这种行为。

例如，有些可能代表不同的版本。这在 A/B 测试中可能有用，我们可能希望在其中配置基于不同服务版本的流量百分比路由，或指引从内部用户到特定实例集的流量。

使用VirtualService，我们可以为一个或多个主机名指定流量行为。
在VirtualService中使用路由规则，告诉 Envoy 如何发送VirtualService的流量到适当的目标。
路由目标地址可以是同一服务的不同版本，也可以是完全不同的服务。

一个典型的场景是将流量发送到被指定为服务子集的服务的不同版本。

客户端将 VirtualService 视为一个统一入口，将请求发送至 VirtualService 主机，然后 Envoy 根据 VirtualService 规则把流量路由到不同的版本。

例如，“20% 的调用转到新版本”或“将这些用户的调用转到版本 2”。
这其实是一个典型的金丝雀发布，我们可以逐步增加发送到新版本服务的流量百分比。

流量路由完全独立于实例部署，这意味着实现新版本服务的实例可以根据流量的负载来伸缩，完全不影响流量路由。

此外，VirtualService还可以帮助我们：

 - 通过单个VirtualService处理多个应用程序服务，我们可以配置一个VirtualService处理特定命名空间中的所有服务。映射单一的VirtualService到多个“真实”服务特别有用，可以在不需要客户适应转换的情况下，将单体应用转换为微服务构建的复合应用系统。我们的路由规则可以指定为“对这些 monolith.com 的 URI 调用转到microservice A”等等。。
 - 和网关整合并配置流量规则来控制出入流量。

在某些情况下，我们还需要配置DestinationRule来使用这些特性，因为这是指定服务子集的地方。

在一个单独的对象中指定服务子集和其它特定目标策略，有利于在VirtualService之间更简洁地重用这些规则。在后续的内容中，我们可以找到更多关于DestinationRule的内容。


### VirtualService 示例

下面，我们来看一个 VirtualService 的示例，它的功能是根据请求是否来自特定的用户，把它们路由到服务的不同版本:

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

下面，我们对一些核心点进行分析，

首先来看一下 **hosts** 字段：

hosts 字段用于列举VirtualService的主机: 即用户指定的目标或是路由规则设定的目标。这是客户端向服务发送请求时使用的一个或多个地址。

```yaml
hosts:
- reviews
```

VirtualService主机名可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或显式地指向一个完全限定域名（FQDN）。

也可以使用通配符（“*”）前缀，让我们创建一组匹配所有服务的路由规则。

VirtualService的 hosts 字段实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址。这让我们可以为没有路由到网格内部的虚拟主机建模。

接下来，我们来了解一下 **路由规则**:

在 http 字段包含了VirtualService的路由规则，这些路由规则用来描述匹配条件和路由行为，它们把原本发送到 hosts 字段的 HTTP/1.1、HTTP2 和 gRPC 等流量发送到规则中指定的目标上。
（我们也可以用 tcp 和 tls 片段为 TCP 和未终止的 TLS 流量设置路由规则）。

下面，我们来看一下匹配规则是如何定义的。

示例中的第一个路由规则有一个条件，因此以 match 字段开始。
在本例中，我们希望此路由应用于来自 ”jason“ 用户的所有请求，所以使用 headers、end-user 和 exact 字段选择适当的请求。

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

route 部分的 destination 字段指定了符合此条件的流量的实际目标地址。

与VirtualService的 hosts 不同，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。

它可以是一个有代理的服务网格，或者是一个通过服务入口被添加进来的非网格服务。

本示例运行在 Kubernetes 环境中，host 名为一个 Kubernetes 服务名：

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

请注意，在该示例和本页其它示例中，为了简单，我们使用 Kubernetes 的短名称设置 destination 的 host。
在评估此规则时，Istio 会添加一个基于VirtualService命名空间的域后缀，这个VirtualService包含要获取主机的完全限定名的路由规则。
在我们的示例中使用短名称也意味着我们可以复制并在任何喜欢的命名空间中尝试它们。

Ps: 只有在目标主机和VirtualService位于相同的 Kubernetes 命名空间时才可以使用这样的短名称。
因为使用 Kubernetes 的短名称容易导致配置出错，我们建议我们在生产环境中指定完全限定的主机名。

destination 部分还指定了 Kubernetes 服务的子集，将符合此规则条件的请求转入其中。 在本例中子集名称是 v2。
我们可以在后续的目标规则中看到如何定义服务子集。

从上面的示例中，我们其实可以看到，在 http 字段下，我们可以包含多个路由规则，那么具体应该哪一个路由规则生效呢？它们的优先级是什么样的呢？

**路由规则按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。**

本示例中，所有不满足第一个路由规则的流量均会流向default目标，default 目标本质上就是一个没有 match 条件的规则。

```yaml
- route:
  - destination:
      host: reviews
      subset: v3
```

如上述示例所示，我们建议提供一个默认的“无条件”或基于权重的规则作为每一个虚拟服务的最后一条规则，从而确保流经虚拟服务的流量至少能够匹配一条路由规则。

除了根据 Header 字段来进行流量调度外，我们还可以在流量端口、URI等内容上设置匹配条件，例如：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
```

如上示例中，这个虚拟服务让用户发送请求到两个独立的服务：ratings 和 reviews，就好像它们是 http://bookinfo.com/ 这个更大的虚拟服务的一部分。
虚拟服务规则根据请求的 URI 匹配流量来指向适当服务的请求。

在一些match表达式中，除了精准匹配之外，还可以使用前缀匹配或正则匹配等多种匹配方式。

我们可以使用 AND 向同一个 match 块添加多个匹配条件，或者使用 OR 向同一个规则添加多个 match 块。
对于任何一个虚拟服务也可以有多个路由规则。这些组合可以帮助你在单个虚拟服务中使路由条件变得非常灵活和强大。

另外，我们还可以使用匹配条件我们可以按百分比”权重“分发请求。这在 A/B 测试和金丝雀发布中非常有用：

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

我们也可以使用路由规则在流量上执行一些操作，例如：

 - 添加或删除 header。
 - 重写 URL。
 - 为调用这一目标地址的请求设置重试策略。

想要进一步了解如果使用这些功能，可以参考 HTTPRoute 。

## DestinationRule

与虚拟服务一样，目标规则也是 Istio 流量路由功能的关键部分。
我们可以将虚拟服务视为将流量如何路由到给定目标地址，然后使用目标规则来配置该目标的流量。
在评估虚拟服务路由规则之后，目标规则将应用于流量的“真实”目标地址。

特别是，我们可以使用目标规则来指定命名的服务子集，例如按版本为所有给定服务的实例分组。
然后可以在虚拟服务的路由规则中使用这些服务子集来控制到服务不同实例的流量。

目标规则还允许我们在调用整个目的地服务或特定服务子集时定制 Envoy 的流量策略，比如我们喜欢的负载均衡模型、TLS 安全模式或熔断器设置。

默认情况下，Istio 使用轮询的负载均衡策略，实例池中的每个实例依次获取请求。

### 负载均衡策略

Istio 同时支持如下的负载均衡模型，可以在 DestinationRule 中为流向某个特定服务或服务子集的流量指定这些负载均衡模型：

 - 随机: 请求以随机的方式转到池中的实例。
 - 权重: 请求根据指定的百分比转到实例。
 - 最少请求: 请求被转到最少被访问的实例。

### DestinationRule 示例

在下面的示例中，目标规则为 my-svc 目标服务配置了 3 个具有不同负载均衡策略的子集：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

其中，每个子集都是基于一个或多个 labels 定义的，在 Kubernetes 中它是附加到像 Pod 这种对象上的键/值对。
这些标签应用于 Kubernetes 服务的 Deployment 并作为 metadata 来识别不同的版本。

除了定义子集之外，目标规则对于所有子集都有默认的流量策略，而对于有定义的子集，则有特定于子集的策略覆盖它。
例如，定义在 subsets 上的默认策略，为 v1 和 v3 子集设置了一个简单的随机负载均衡器。在 v2 策略中，轮询负载均衡器被指定在相应的子集字段上。

## 网关

使用网关可以为网格来管理入站和出站流量，从而可以管理进入或离开网格的流量。
网关配置被用于运行在网格边界的独立 Envoy 代理，而不是服务工作负载的 sidecar 代理。

与 Kubernetes Ingress API 这种控制进入系统流量的机制不同，Istio 网关让我们充分利用流量路由的强大能力和灵活性。
其中，主要的原因是Istio的网关资源不仅仅生效于应用层流量（L7），同时还可以配置4-6层的负载均衡属性。

网关主要用于管理进入的流量，但我们也可以配置出口网关。
出口网关让您为离开网格的流量配置一个专用的出口节点，这可以限制哪些服务可以或应该访问外部网络，或者启用出口流量安全控制为您的网格添加安全性。
当然，您也可以使用网关配置一个纯粹的内部代理。

Istio 提供了一些预先配置好的网关代理部署（istio-ingressgateway 和 istio-egressgateway）供您使用，在demo的安装中，它们都已经部署好了。

### 网关示例

下面的示例展示了一个外部 HTTPS 入口流量的网关配置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

这个网关配置让 HTTPS 流量从 ext-host.example.com 通过 443 端口流入网格，但没有为请求指定任何路由规则。
为想要工作的网关指定路由，您必须把网关绑定到虚拟服务上。
正如下面的示例所示，使用虚拟服务的 gateways 字段进行设置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
    - ext-host-gwy
```

然后就可以为出口流量配置带有路由规则的虚拟服务。


## Service Entries

使用服务入口可以向Istio 内部维护的服务注册中心添加一个请求入口。

添加了服务入口后，Envoy 代理可以向服务发送流量，就好像它是网格内部的服务一样。

配置服务入口允许您管理运行在网格外的服务的流量，它包括以下几种能力：

 - 为外部目标 redirect 和转发请求，例如来自 web 端的 API 调用，或者流向遗留老系统的服务。
 - 为外部目标定义重试、超时和故障注入策略。
 - 添加一个运行在虚拟机的服务来扩展您的网格。
 - 从逻辑上添加来自不同集群的服务到网格，在 Kubernetes 上实现一个多集群 Istio 网格。

您不需要为网格服务要使用的每个外部服务都添加服务入口。
默认情况下，Istio 配置 Envoy 代理将请求传递给未知服务。
但是，您不能使用 Istio 的特性来控制没有在网格中注册的目标流量。

### 服务入口示例

下面示例的 mesh-external 服务入口将 ext-resource 外部依赖项添加到 Istio 的服务注册中心：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

您可以使用 hosts 字段指定外部资源，可以使用完全限定名或通配符作为前缀域名。

您可以配置虚拟服务和目标规则，以更细粒度的方式控制到服务入口的流量，这与网格中的任何其他服务配置流量的方式相同。

例如，下面的目标规则配置流量路由以使用双向 TLS 来保护到 ext-svc.example.com 外部服务的连接，我们使用服务入口配置了该外部服务：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

## SideCar

默认情况下，Istio 让每个 Envoy 代理都可以访问来自和它关联的工作负载的所有端口的请求，然后转发到对应的工作负载。
您可以使用 sidecar 配置去做下面的事情：

 - 微调 Envoy 代理接受的端口和协议集。
 - 限制 Envoy 代理可以访问的服务集合。

您可能希望在较庞大的应用程序中限制这样的 sidecar 可达性，配置每个代理能访问网格中的任意服务可能会因为高内存使用量而影响网格的性能。

您可以指定将 sidecar 配置应用于特定命名空间中的所有工作负载，或者使用 workloadSelector 选择特定的工作负载。

例如，下面的 sidecar 配置将 bookinfo 命名空间中的所有服务配置为仅能访问运行在相同命名空间和 Istio 控制平面中的服务（目前需要使用 Istio 的策略和遥测功能）：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

## 网络弹性与测试

除了为您的网格导流之外，Istio 还提供了可选的故障恢复和故障注入功能，您可以在运行时动态配置这些功能。
使用这些特性可以让您的应用程序运行稳定，确保服务网格能够容忍故障节点，并防止局部故障级联影响到其他节点。

### 超时

超时是 Envoy 代理等待来自给定服务的答复的时间量，以确保服务不会因为等待答复而无限期的挂起，并在可预测的时间范围内返回调用成功或失败。
HTTP 请求的默认超时时间是 15 秒，这意味着如果服务在 15 秒内没有响应，调用将失败。

对于某些应用程序和服务，Istio 的缺省超时可能不合适。
例如，超时太长可能会由于等待失败服务的回复而导致过度的延迟；而超时过短则可能在等待涉及多个服务返回的操作时触发不必要地失败。

为了找到并使用最佳超时设置，Istio 允许您使用虚拟服务按服务轻松地动态调整超时，而不必修改您的业务代码。

下面的示例是一个虚拟服务，它对 ratings 服务的 v1 子集的调用指定 10 秒超时：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

### 重试

重试设置指定如果首次调用失败，Envoy 代理尝试连接服务的最大次数。
重试通过确保请求不会因为偶发的抖动而失败，从而可以提高服务可用性和应用程序的性能。
重试之间的间隔（25ms+）是可变的，并由 Istio 自动确定，从而防止被调用服务被请求淹没。
HTTP 请求的默认重试行为是在返回错误之前重试两次。

与超时一样，Istio 默认的重试行为在延迟方面可能不适合您的应用程序需求（对失败的服务进行过多的重试会降低速度）或可用性。
您可以在虚拟服务中按服务调整重试设置，而不必修改业务代码。
您还可以通过添加每次重试的超时来进一步细化重试行为，并指定每次重试都试图成功连接到服务所等待的时间量。

下面的示例配置了在首次调用失败后最多重试 3 次来连接到服务子集，每个重试都有 2 秒的超时。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

### 熔断

熔断是 Istio 为创建具有弹性的微服务应用提供的另一个有用的机制。
在熔断中，可以设置一个对服务中的单个主机调用的限制，例如并发连接的数量或对该主机调用失败的次数。
一旦限制被触发，熔断器就会“跳闸”并停止连接到该主机。使用熔断模式可以快速失败而不必让客户端尝试连接到过载或有故障的主机。

熔断适用于在负载均衡池中的“真实”网格目标地址，您可以在目标规则中配置熔断器阈值，让配置适用于服务中的每个主机。
下面的示例将 v1 子集的reviews服务工作负载的并发连接数限制为 100：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
```

### 故障注入

在配置了网络，包括故障恢复策略之后，可以使用 Istio 的故障注入机制来为整个应用程序测试故障恢复能力。
故障注入是一种将错误引入系统以确保系统能够承受并从错误条件中恢复的测试方法。
使用故障注入特别有用，能确保故障恢复策略不至于不兼容或者太严格，这会导致关键服务不可用。

与其他错误注入机制（如延迟数据包或在网络层杀掉 Pod）不同，Istio 允许在应用层注入错误。
这使您可以注入更多相关的故障，例如 HTTP 错误码，以获得更多相关的结果。

您可以注入两种故障，它们都使用虚拟服务配置：

 - 延迟：延迟是时间故障。它们模拟增加的网络延迟或一个超载的上游服务。
 - 终止：终止是崩溃失败。他们模仿上游服务的失败。终止通常以 HTTP 错误码或 TCP 连接失败的形式出现。

例如，下面的虚拟服务为千分之一的访问 ratings 服务的请求配置了一个 5 秒的延迟：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

### Istio 和应用程序同时工作

Istio 故障恢复功能对应用程序来说是完全透明的。
在返回响应之前，应用程序不知道 Envoy sidecar 代理是否正在处理被调用服务的故障。
这意味着，如果在应用程序代码中设置了故障恢复策略，那么您需要记住这两个策略都是独立工作的，否则会发生冲突。

例如，假设您设置了两个超时，一个在虚拟服务中配置，另一个在应用程序中配置。应用程序为服务的 API 调用设置了 2 秒超时。
而您在虚拟服务中配置了一个 3 秒超时和重试。
在这种情况下，应用程序的超时会先生效，因此 Envoy 的超时和重试尝试会失效。

虽然 Istio 故障恢复特性提高了网格中服务的可靠性和可用性，但应用程序必须处理故障或错误并采取适当的回退操作。
例如，当负载均衡中的所有实例都失败时，Envoy 返回一个HTTP 503代码。应用程序必须实现回退逻辑来处理HTTP 503错误代码。

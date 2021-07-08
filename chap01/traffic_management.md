# Istio 流量管理

Istio 的流量路由规则可以让您很容易的控制服务之间的流量和 API 调用。

Istio 简化了服务级别属性的配置，比如熔断器、超时和重试，并且能轻松的设置重要的任务，如 A/B 测试、金丝雀发布、基于流量百分比切分的概率发布等。

它还提供了开箱即用的故障恢复特性，有助于增强应用的健壮性，从而更好地应对被依赖的服务或网络发生故障的情况。

Istio 的流量管理模型源于和服务一起部署的 Envoy 代理。

网格内服务发送和接收的所有流量（data plane流量）都经由 Envoy 代理，这让控制网格内的流量变得异常简单，而且不需要对服务做任何的更改。

本节中描述的功能特性，如果您对它们是如何工作的感兴趣的话，可以在 [架构概述](../chap05/architecture.md) 中找到关于 Istio 的流量管理实现的更多信息。

本部分只介绍 Istio 的流量管理特性。

## 概述

为了在服务网格中导流，Istio 需要知道所有的 endpoint 在哪和它属于哪个服务。

为了定位到service registry(服务注册中心)，Istio 会连接到一个服务发现系统。

例如，如果您在 Kubernetes 集群上安装了 Istio，那么它将自动检测该集群中的服务和 endpoint。

使用此服务注册中心，Envoy 代理可以将流量定向到相关服务。

大多数基于微服务的应用程序，每个服务的工作负载都有多个实例来处理流量，称为负载均衡池。
默认情况下，Envoy 代理基于轮询调度模型在服务的负载均衡池内分发流量，按顺序将请求发送给池中每个成员，一旦所有服务实例均接收过一次请求后，重新回到第一个池成员。

Istio 基本的服务发现和负载均衡能力为您提供了一个可用的服务网格，但它能做到的远比这多的多。

在许多情况下，您可能希望对网格的流量情况进行更细粒度的控制。

作为 A/B 测试的一部分，您可能想将特定百分比的流量定向到新版本的服务，或者为特定的服务实例子集应用不同的负载均衡策略。

您可能还想对进出网格的流量应用特殊的规则，或者将网格的外部依赖项添加到服务注册中心。

通过使用 Istio 的流量管理 API 将流量配置添加到 Istio，就可以完成所有这些甚至更多的工作。

和其他 Istio 配置一样，这些 API 也使用 Kubernetes 的自定义资源定义（CRDs）来声明，您可以像示例中看到的那样使用 YAML 进行配置。

接下来，我们将会对 Istio 流量管理中用到的这些 API 资源对象依次进行讲解和说明。

## VirtualService

VirtualService 和 DestinationRule 是 Istio 流量路由功能中最核心的对象。

VirtualService 让您配置如何在服务网格内将请求路由到服务，这基于 Istio 和平台提供的基本的连通性和服务发现能力。

每个 VirtualService 包含一组路由规则，Istio 按顺序去评估它们，Istio 将每个给定的请求匹配到VirtualService指定的实际目标地址。

### 为什么需要 VirtualService

VirtualService 在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用。

通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。

VirtualService同时提供了丰富的方式，为发送至这些工作负载的流量指定不同的路由规则。

为什么 VirtualService 如此有用？

就像在之前的介绍中所说，如果没有VirtualService，Envoy 会在所有的服务实例中使用轮询的负载均衡策略分发请求。 您可以用您对工作负载的了解来改善这种行为。

例如，有些可能代表不同的版本。这在 A/B 测试中可能有用，您可能希望在其中配置基于不同服务版本的流量百分比路由，或指引从内部用户到特定实例集的流量。

使用VirtualService，您可以为一个或多个主机名指定流量行为。
在VirtualService中使用路由规则，告诉 Envoy 如何发送VirtualService的流量到适当的目标。
路由目标地址可以是同一服务的不同版本，也可以是完全不同的服务。

一个典型的场景是将流量发送到被指定为服务子集的服务的不同版本。

客户端将 VirtualService 视为一个统一入口，将请求发送至 VirtualService 主机，然后 Envoy 根据 VirtualService 规则把流量路由到不同的版本。

例如，“20% 的调用转到新版本”或“将这些用户的调用转到版本 2”。
这其实是一个典型的金丝雀发布，我们可以逐步增加发送到新版本服务的流量百分比。

流量路由完全独立于实例部署，这意味着实现新版本服务的实例可以根据流量的负载来伸缩，完全不影响流量路由。

此外，VirtualService还可以帮助您：

 - 通过单个VirtualService处理多个应用程序服务，我们可以配置一个VirtualService处理特定命名空间中的所有服务。映射单一的VirtualService到多个“真实”服务特别有用，可以在不需要客户适应转换的情况下，将单体应用转换为微服务构建的复合应用系统。您的路由规则可以指定为“对这些 monolith.com 的 URI 调用转到microservice A”等等。。
 - 和网关整合并配置流量规则来控制出入流量。

在某些情况下，您还需要配置DestinationRule来使用这些特性，因为这是指定服务子集的地方。

在一个单独的对象中指定服务子集和其它特定目标策略，有利于在VirtualService之间更简洁地重用这些规则。在后续的内容中，您可以找到更多关于DestinationRule的内容。


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

也可以使用通配符（“*”）前缀，让您创建一组匹配所有服务的路由规则。

VirtualService的 hosts 字段实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址。这让您可以为没有路由到网格内部的虚拟主机建模。

接下来，我们来了解一下 **路由规则**:

在 http 字段包含了VirtualService的路由规则，这些路由规则用来描述匹配条件和路由行为，它们把原本发送到 hosts 字段的 HTTP/1.1、HTTP2 和 gRPC 等流量发送到规则中指定的目标上。
（您也可以用 tcp 和 tls 片段为 TCP 和未终止的 TLS 流量设置路由规则）。

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
在我们的示例中使用短名称也意味着您可以复制并在任何喜欢的命名空间中尝试它们。

Ps: 只有在目标主机和VirtualService位于相同的 Kubernetes 命名空间时才可以使用这样的短名称。
因为使用 Kubernetes 的短名称容易导致配置出错，我们建议您在生产环境中指定完全限定的主机名。

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

您可以使用 AND 向同一个 match 块添加多个匹配条件，或者使用 OR 向同一个规则添加多个 match 块。
对于任何一个虚拟服务也可以有多个路由规则。这些组合可以帮助你在单个虚拟服务中使路由条件变得非常灵活和强大。

另外，我们还可以使用匹配条件您可以按百分比”权重“分发请求。这在 A/B 测试和金丝雀发布中非常有用：

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

您也可以使用路由规则在流量上执行一些操作，例如：

 - 添加或删除 header。
 - 重写 URL。
 - 为调用这一目标地址的请求设置重试策略。

想要进一步了解如果使用这些功能，可以参考 HTTPRoute 。

## DestinationRule


## Gateway


## Service Entries


## SideCar


## 网络弹性与测试


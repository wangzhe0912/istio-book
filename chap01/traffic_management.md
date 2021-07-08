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

在一个单独的对象中指定服务子集和其它特定目标策略，有利于在虚拟服务之间更简洁地重用这些规则。在后续的内容中，您可以找到更多关于DestinationRule的内容。


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

## DestinationRule


## Gateway


## Service Entries


## SideCar


## 网络弹性与测试


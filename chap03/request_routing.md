# 请求路由

请求路由是指将请求动态路由到微服务的多个不同的版本上。

在之前的 [示例](../chap04/bookinfo.md) 中，我们已经部署完成了 Bookinfo 应用。
下面，将会基于 Bookinfo 应用来演示如果使用 Istio 来控制请求路由。

在之前的示例中，其中一个微服务 reviews 的三个不同版本已经部署并同时运行。
这导致在浏览器中访问 Bookinfo 应用程序的 /productpage 并刷新几次，
有时书评的输出包含星级评分，有时则不包含。
这是因为没有明确的默认服务版本可路由，Istio 将以循环方式将请求路由到所有可用版本。

下面，我们将会在本文中，我们会先将所有流量都路由到 reviews 的 v1 版本，
然后在演示如何根据 HTTP header 的值来进行流量路由。

## 将所有流量都路由到 reviews 的 v1 版本

首先，我们希望将所有访问 reviews 的请求全部路由到 v1 版本时，
我们需要为微服务 reviews 设置一个默认版本的 VirtualService 。
在 VirtualService 中，我们可以指定流量路由到的实例版本。

```shell
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

由于配置传播是最终一致的，因此请等待几秒钟以使 Virtual Service 生效。

可以查询一下看到如下结果：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
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
---
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
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
---
```

此时，我们已将 Istio 配置为路由到 Bookinfo 微服务的 v1 版本，最重要的是 reviews 服务的版本 1。

您可以通过再次刷新 Bookinfo 应用程序的 /productpage 轻松测试新配置。
无论您刷新多少次，页面的评论部分都不会显示评级星标。这是因为您将 Istio 配置为将评论服务的所有流量路由到版本 reviews:v1，
而此版本的服务不访问星级评分服务。

## 根据 HTTP header 的值来进行流量路由

下面，我们来进行一个更加『高级』的功能，更改路由配置，以便将来自特定用户的所有流量路由到特定服务版本。
例如，我们可以将来自名为 Jason 的用户的所有流量将被路由到服务 reviews:v2。

Ps: Istio 对用户身份没有任何特殊的内置机制。
事实上，productpage 服务在所有到 reviews 服务的 HTTP 请求中都增加了一个自定义的 end-user 请求头，从而达到了本例子的效果。

我们可以运行以下命令以启用基于用户的路由：

```yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

其中配置如下:

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
        subset: v1
```

在 Bookinfo 应用程序的 /productpage 上，以用户 jason 身份登录，刷新浏览器。您看到了什么？星级评分显示在每个评论旁边。

然后以其他用户身份登录（选择您想要的任何名称），
刷新浏览器。现在星星消失了。这是因为除了 Jason 之外，所有用户的流量都被路由到 reviews:v1。

## 原理概述

在此任务中，您首先使用 Istio 将 100% 的请求流量都路由到了 Bookinfo 服务的 v1 版本。
然后设置了一条路由规则，它根据 productpage 服务发起的请求中的 end-user 自定义请求头内容，
选择性地将特定的流量路由到了 reviews 服务的 v2 版本。

请注意，Kubernetes 中的服务，如本任务中使用的 Bookinfo 服务，必须遵守某些特定限制，才能利用到 Istio 的 L7 路由特性优势。
参考 [Pod 和 Service 需求](../chap05/requirements.md) 了解详情。

在下一节流量转移任务中，您将按照在此处学习到的相同的基本模式来配置路由规则，以逐步将流量从服务的一个版本迁移到另一个版本。

## 环境恢复

该实验完成后，为了恢复初识状态，我们可以删除对应的 VirtualService 。

```shell
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

再次刷新页面，可以看到评分显示的状态又在不断的变化了。

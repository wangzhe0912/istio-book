# 书店应用 Istio 化

在本文中，我们将会以一个示例项目为例，来演示多种 Istio 特性功能的应用。

## 项目概述

在该示例项目中，包含四个单独的微服务。
这个应用模仿在线书店的一个分类，显示一本书的信息。 页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

 - productpage: 这个微服务会调用 details 和 reviews 两个微服务，用来生成页面。
 - details: 这个微服务中包含了书籍的信息。
 - reviews: 这个微服务中包含了书籍相关的评论。它还会调用 ratings 微服务。
 - ratings: 这个微服务中包含了由书籍评价组成的评级信息。

其中，reviews 微服务有 3 个版本：

 - v1 版本不会调用 ratings 服务。
 - v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
 - v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

下图展示了这个应用的端到端架构：

![bookinfo1](./pictures/bookinfo1.svg)

Bookinfo 应用中的几个微服务是由不同的语言编写的。
这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 reviews 服务具有多个版本。

在正式开始之前，首先你需要完整搭建一套 K8s 环境并部署完成 Istio 。

## 应用部署

要在 Istio 中运行这一应用，无需对应用自身做出任何改变。
您只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。
最终的部署结果将如下图所示：

![bookinfo2](./pictures/bookinfo2.svg)

所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，
这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。

具体来说：

Step1: 为 default 命名空间设置为自动注入 Istio 。

```shell
kubectl label namespace default istio-injection=enabled
```

Step2: 部署 Bookinfo 应用：

```shell
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

上面的命令会启动全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）。

Ps: 在实际部署中，微服务版本的启动过程需要持续一段时间，并不是同时完成的。

Step3: 确认所有的服务和 Pod 都已经正确的定义和启动：

```shell
kubectl get services
kubectl get pods
```

Step4: 要确认 Bookinfo 应用是否正在运行，请在某个 Pod 中用 curl 命令对应用发送请求，例如 ratings：

```shell
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>
```

Step5: 确定 Ingress 的 IP 和端口

现在 Bookinfo 服务启动并运行中，您需要使应用程序可以从外部访问 Kubernetes 集群，例如使用浏览器。可以用 Istio Gateway 来实现这个目标。

```shell
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway
```

查询网关的 INGRESS_HOST 和 INGRESS_PORT 变量，确认并设置：

```shell
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
echo $INGRESS_HOST
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
echo $INGRESS_PORT
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
echo $SECURE_INGRESS_PORT
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "$GATEWAY_URL"
```

Step6: 从集群外部访问应用

当我们得到 $GATEWAY_URL 之后，就可以通过浏览器从集群外部来访问集群资源了。

我们可以用浏览器打开网址 http://$GATEWAY_URL/productpage ，来浏览应用的 Web 页面。如果刷新几次应用的页面，
就会看到 productpage 页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。
reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。

## 应用默认目标规则

在使用 Istio 控制 Bookinfo 版本路由之前，您需要在目标规则中定义好可用的版本，命名为 subsets 。

运行以下命令为 Bookinfo 服务创建的默认的目标规则：

```shell
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

等待几秒钟，以使目标规则生效。

可以使用以下命令查看目标规则：

```yaml
kubectl get destinationrules -o yaml
```

至此为止，我们就完成了 Istio 实验的基本环境准备工作，接下来，我们就可以使用这一应用来体验 Istio 的特性了，其中包括了流量的路由、错误注入、速率限制等。

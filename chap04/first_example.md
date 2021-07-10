# 快速入门

在本文中，我们将会以一个非常简单的入门示例来演示关于 VirtualService 和 DestinationRule 相关的基本概念。

## 基础环境准备

首先，为了演示该实例，我们首先来创建一个新的 namespace 并将该 namespace 设置为自动注入 sidecar 。

```shell
kubectl create ns books
kubectl label namespace books istio-injection=enabled
```

## 业务服务部署

下面，我们来部署对应的业务容器，该业务容器是一个 Flask 服务，可以通过 url 查询它对应的环境变量。

其中，业务容器的镜像为: dustise/flaskapp

下面，我们准备一个 `flaskapp.yaml` 的文件，该文件中包含了 flaskdemo 的 deployment 定义与对应的 service 定义。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flaskapp
  labels:
    app: flaskapp
spec:
  selector:
    app: flaskapp
  ports:
    - name: http
      port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskapp-v1
spec:
  selector:
    matchLabels:
      app: flaskapp
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: flaskapp
        version: v1
    spec:
      containers:
      - name: flaskapp
        image: dustise/flaskapp
        imagePullPolicy: IfNotPresent
        env:
        - name: version
          value: v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskapp-v2
spec:
  selector:
    matchLabels:
      app: flaskapp
      version: v2
  replicas: 1
  template:
    metadata:
      labels:
        app: flaskapp
        version: v2
    spec:
      containers:
      - name: flaskapp
        image: dustise/flaskapp
        imagePullPolicy: IfNotPresent
        env:
        - name: version
          value: v2
```

然后创建业务容器：

```shell
kubectl apply -f flaskapp.yaml -n books
```

下面，有个业务容器之后，我们再来创建一个客户端容器，用于向业务容器发起访问。

客户端容器的镜像非常简单，只安装了一个 curl 等相关的指令，镜像为: dustise/sleep

下面，我们准备了一个 `client.yaml` 的文件，该文件包含了对应客户端文件的 deployment 的定义：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  selector:
    matchLabels:
      app: sleep
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
        version: v1
    spec:
      containers:
      - name: sleep
        image: dustise/sleep
        imagePullPolicy: IfNotPresent
```

然后创建客户端部署对象：

```shell
kubectl apply -f client.yaml -n books
```

下面，我们就可以进入客户端的容器了：

```shell
kubectl exec -it sleep-7575b5d557-n68b4 -c sleep bash -n books
```

然后在容器内可以执行如下命令来访问业务容器：

```shell
for i in `seq 10`; do http --body http://flaskapp.book/env/version; done
```

此时，查看请求返回的结果，我们应该可以看到，v1 和 v2 的业务容器基本上是均匀被访问，这是 K8s Service 中提供的基本的负载均衡访问的能力。

## 创建 VirtualService 和 DestinationRule

下面，我们就来体验一下 Istio 相关的功能吧，之前的基本概念中，我们已经了解了 DestinationRule 和 VirtualService 的基本概念，
我们就来在实际应用中应用一下吧。

首先，我们来创建 flaskapp 应用的 DestinationRule，它可以将应用分为多个子网，然后在 VirtualService 可以指定到对应的子网请求。

首先，我们准备一个 `destination_rule.yaml` 文件：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: flaskapp
spec:
  host: flaskapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

在该 destination_rule 的配置文件中，我们将对应的 flaskapp 服务下面的 Pod 安装 label 中 version 字段的不同，
分为了 v1 和 v2 两个子集。

下面，我们再来创建一个 `virtual_service.yaml` 文件：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: flaskapp-policy
spec:
  hosts:
  -  flaskapp
  http:
  - route:
    - destination:
        host: flaskapp
        subset: v2
```

通过 VirtualService 的配置，我们可以将该所有请求至 flaskapp Service 的请求，按照对应的规则发送给对应的目的下游。

其中，在上述的规则配置中，我们将 flaskapp Service 的请求全部转发给了 flaskapp 下对应的 v2 子集下的实例。

下面，我们应用一下这两个配置文件：

```shell
kubectl apply -f client.destination_rule -n books
kubectl apply -f virtual_service.yaml -n books
```

## 验证相关的效果

当对应的 VirtualService 和 DestinationRule 的规则都配置完成后，下面，我们就可以来验证一下效果了。

再次在客户端容器内请求业务容器试试：

```shell
for i in `seq 10`; do http --body http://flaskapp.book/env/version; done
```

可以看到此时返回的结果预期应该已经全部都是 v2 了，对！我们的流量控制已经生效了。

下面，我们还可以修改一下 `virtual_service.yaml` 文件的规则，将请求流量都转到 v1 子集上，然后重新 apply 生效：

```shell
kubectl apply -f virtual_service.yaml -n books
```

再次请求一下看看吧，响应是否已经都切回到v1了？

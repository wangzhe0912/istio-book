# DestinationRule 配置详解

## 基本概念

DestinationRule 定义了在路由发生后应用于服务流量的策略。
这些规则指定负载均衡的配置、来自 sidecar 的连接池大小和异常检测设置，以检测和从负载均衡池中驱逐不健康的实例。

例如，ratings 服务的简单负载均衡策略如下所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

可以通过定义命名子集并覆盖在服务级别指定的设置来指定特定于版本的策略。

以下规则对所有流向名为 testversion 的子集的流量使用循环负载均衡策略，该子集由带有标签 (version:v3) 的端点（例如 pod）组成。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

注意：在路由规则明确将流量发送到该子集之前，为子集指定的策略不会生效。

流量策略也可以针对特定端口进行定制。以下规则对到端口 80 的所有流量使用最少连接负载均衡策略，而对到端口 9080 的流量使用循环负载均衡设置。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings-port
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy: # Apply to all ports
    portLevelSettings:
    - port:
        number: 80
      loadBalancer:
        simple: LEAST_CONN
    - port:
        number: 9080
      loadBalancer:
        simple: ROUND_ROBIN
```

## DestinationRule

DestinationRule 定义了在路由发生后应用于服务流量的策略。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|host|string|来自服务注册中心的服务名称。服务名称从平台的服务注册表（例如 Kubernetes 服务、Consul 服务等）和 ServiceEntries 声明的host中查找，找不到时则忽略该规则。host字段适用于HTTP和TCP服务。|是|
|trafficPolicy|TrafficPolicy|要应用的流量策略（负载均衡策略、连接池大小、异常值检测）|否|
|subsets|Subset[]|代表服务的各个版本的一个或多个命名集。也可以在子集级别设置流量策略。|否|
|exportTo|string[]|将此DesinationRule导出到的namespace列表。应用到服务的DesinationRule的解析发生在namespace层次结构的上下文中。导出DesinationRule允许将其包含在其他namespace中服务的解析层次结构中。此功能为服务所有者和网格管理员提供了一种机制，以控制跨namespace边界的DesinationRule的可见性。如果未指定namespace，则默认情况下将DesinationRule导出到所有namespace。其中，.导出到声明DesinationRule的所在namespace，*导出到所有namespace，在当前版本中，exportTo 值被限制为“.”。或“*”。|否|


## Subset

服务端点的子集。
子集可用于 A/B 测试等场景，或路由到服务的特定版本。
此外，可以在子集级别覆盖在服务级别定义的流量策略。
以下规则对所有流向名为 testversion 的子集的流量使用循环负载均衡策略，该子集由带有标签 (version:v3) 的端点（例如 pod）组成。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

注意：在路由规则明确将流量发送到该子集之前，为子集指定的策略不会生效。

通常需要一个或多个标签来标识子集目的地，但是，当相应的 DestinationRule 表示支持多个 SNI 主机的主机（例如，出口网关）时，
没有标签的子集可能是有意义的。在这种情况下，可以使用带有 TLSSettings 的流量策略来识别与命名子集对应的特定 SNI 主机。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|name|string|子集的名称，服务名称和子集名称可用于路由规则中的流量拆分。|是|
|labels|map<string,string>|标签在服务注册表中的服务端点上应用过滤器，即对 Pod 属性进行的过滤。|否|
|trafficPolicy|TrafficPolicy|适用于该子集的流量策略。子集继承在 DestinationRule 级别指定的流量策略。在子集级别指定的设置将覆盖在 DestinationRule 级别指定的相应设置。|否|

## TrafficPolicy




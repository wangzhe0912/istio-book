# VirtualService 配置详解

## 基本概念

**Service** 是绑定到服务注册表中唯一名称的应用程序行为单元。
Service 由在 pod、容器、VM 等上运行的工作负载实例实现的多个网络端点组成。

**Service Version(称之为 subset)** 是指在持续部署场景中，对于给定的服务，可以有不同的实例子集，它们运行着的应用程序二进制文件的版本可能不同。
例如，它们提供的 API 的版本可能不同，它们也可以是对同一服务的迭代更改，部署在不同的环境（生产、暂存、开发等）中。
其中，A/B 测试、金丝雀发布等常用场景其实就是利用着相关的特性。
在 VirtualService 中，可以通过各种规则（例如url, headers）等来决定访问具体不同版本的权重。
此外，对于每个 VirtualService 而言，默认情况下会将请求发给它的所有实例。

**Source** 是指调用服务的下游客户端。

**Host** 是指客户端尝试连接到服务时使用的地址。

**Access model** 下游服务在访问目标时，仅仅需要指标目标服务的地址（Host），而不需要了解单个服务版本（Subset），
而具体请求下发到哪个版本的实例上面，则是有 Sidecar 来进行决策，从而使得应用程序代码能够将自身与依赖服务的演变进行解耦。

VirtualService 定义了一组要在访问 host 时应用的流量路由规则，每个路由规则定义了特定协议流量的匹配规则。
如果流量的匹配规则满足的话，则将其发送到配置中定义的目标服务（或其子集/版本）中。

请求来源也可以在路由规则中进行条件匹配，通过这一机制，可以实现为特定客户端定制路由机制。

以下 Kubernetes 示例默认将所有 HTTP 流量路由到带有标签“version: v1”的评论服务的 pod。
此外，url以 `/wpcatalog/` 或 `/consumercatalog/` 开头的 HTTP 请求将被重写为 `/newcatalog` 并发送到标签为“version: v2”的 pod。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - name: "reviews-v2-routes"
    match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
```

其中，路由目的地的子集/版本通过对命名服务子集的引用来标识，该子集必须在相应的 **DestinationRule** 中声明，示例如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.prod.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

下面，我们来对其中的一些核心配置字段的配置信息来依次进行详细的说明。

## VirtualService

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|hosts|string[]|流量的目标主机，可以是带有通配符前缀的DNS名称，也可以是IP地址。根据所在平台情况，还可能使用短名称来代替FQDN。这种场景下，短名称到 FQDN 的具体转换过程是要靠下层平台完成的。一个 VirtualService 中可以用于控制多个 HTTP 和 TCP 端口的流量属性，反之，一个目标地址的流量属性也可以同时在多个 VirtualService 中进行定义。对于K8s用户而言，当使用短服务名（如reviews，而不是reviews.default.svc.cluster.local）时，Istio会将短服务名与 VirtualService 定义所属的namespace来进行拼接，而不是K8s Service本身所属的namespace。为了避免潜在的错误配置，建议始终使用完全限定域名而不是短名称。hosts 字段适用于 HTTP 和 TCP 服务。网格内的服务，即在服务注册表中找到的服务，必须始终使用它们的字母数字名称来引用。 IP 地址只允许用于通过网关定义的服务。|是|
|gateways|string[]|VirtualService 对象可以用于网格中的 Sidecar，也可以用于一个或多个 Gateway。这里公开的选择条件可以在协议相关的路由过滤条件中进行覆盖。保留字 mesh 用来指代网格中的所有 Sidecar。当这一字段被省略时，就会使用缺省值（mesh），也就是针对网格中的所有 Sidecar 生效。如果提供了 gateways 字段，这一规则就只会应用到声明的 Gateway 之中。要让规则同时对 Gateway 和网格内服务生效，需要显式的将 mesh 加入 gateways 列表。|否|
|http|HTTPRoute[]|HTTP 流量规则的有序列表。这个列表对名称前缀为 http-、http2-、grpc- 的服务端口，或者协议为 HTTP、HTTP2、GRPC 以及终结的 TLS，另外还有使用 HTTP、HTTP2 以及 GRPC 协议的 ServiceEntry 都是有效的。进入流量会使用匹配到的第一条规则。|否|
|tls|TLSRoute[]|一个有序列表，对应的是透传 TLS 和 HTTPS 流量。路由过程通常利用 ClientHello 消息中的 SNI 来完成。TLS 路由通常应用在 https-、tls- 前缀的平台服务端口，或者经 Gateway 透传的 HTTPS、TLS 协议端口，以及使用 HTTPS 或者 TLS 协议的 ServiceEntry 端口上。注意：没有关联 VirtualService 的 https- 或者 tls- 端口流量会被视为透传 TCP 流量。|否|
|tcp|TCPRoute[]|不透明 TCP 流量的路由规则的有序列表。 TCP 路由将应用于非 HTTP 或 TLS 端口的任何端口。使用匹配传入请求的第一个规则。|否|
|exportTo|string[]|此VirtualService可见的namespace列表。导出VirtualService允许它被其他namespace中定义的sidecar和网关使用。此功能为服务所有者和网格管理员提供了一种机制来控制跨namespace边界的VirtualService的可见性。如果未指定namespace，则默认情况下将VirtualService导出到所有namespace。设置为.表示导出到声明VirtualService的相同namespace。类似地，值“*”被保留并定义到所有namespace的导出。注意：在当前版本中，exportTo 值被限制为“.”。或“*”（即当前namespace或所有namespace）。|否|

## HTTPRoute

描述路由 HTTP/1.1、HTTP2 和 gRPC 流量的匹配条件和操作。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|name|string|为调试目的分配给路由的名称。路由的名称将与匹配的名称连接，并将记录在与此路由/匹配匹配的请求的访问日志中。|否|
|match|HTTPMatchRequest[]|匹配规则要满足的匹配条件。单个匹配块内的所有条件都具有 AND 语义，而匹配块列表具有 OR 语义。|否|
|route|HTTPRouteDestination[]|http 规则可以重定向或转发（默认）流量。转发目标可以是服务的多个版本之一。与服务版本相关的权重决定了它接收的流量比例。|否|
|redirect|HTTPRedirect|http 规则可以重定向或转发（默认）流量。如果规则中指定了流量透传选项，路由/重定向将被忽略。重定向原语可用于将 HTTP 301 重定向发送到不同的 URI 或权限。|否|
|rewrite|HTTPRewrite|重写 HTTP URI 和权限标头。 Rewrite 不能与 Redirect 原语一起使用。在转发之前将执行重写。|否|
|timeout|Duration|HTTP请求的超时时间。|否|
|retries|HTTPRetry|HTTP请求的重试策略。|否|
|fault|HTTPFaultInjection|应用于客户端 HTTP 流量的故障注入策略。请注意，在客户端启用故障时，将不会启用超时或重试。|否|
|mirror|Destination|除了将请求转发到预期目的地之外，还可以将 HTTP 流量镜像到另一个目的地。镜像流量只是会尽量保证镜像流量发送，不会等待镜像集群的响应信息。|否|
|mirrorPercent|uint 32|镜像字段要镜像的流量的百分比。如果此字段不存在，则所有流量 (100%) 都将被镜像。最大值为 100。|否|
|corsPolicy|CorsPolicy|跨源资源共享策略 (CORS)。|否|
|headers|Headers|headers的处理规则。|否|


## HTTPMatchRequest

HttpMatchRequest 指定一组要满足的标准，以便将规则应用于 HTTP 请求。
例如，以下规则将规则限制为仅匹配 URL 路径以 `/ratings/v2/` 开头且请求头中包含 `end-user` 字段的值为 `jason` 的请求。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - headers:
        end-user:
          exact: jason
      uri:
        prefix: "/ratings/v2/"
      ignoreUriCase: true
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
```

**HTTPMatchRequest 字段不允许为空。**

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|name|string|分配给匹配项的名称。匹配的名称将与父路由的名称连接，并将记录在与此路由匹配的请求的访问日志中。|否|
|uri|StringMatch|匹配值的 URI 默认区分大小写，格式如下：exact 表示精确匹配，prefix 表示前缀匹配，regex 表示正则匹配。注意：可以通过 ignore_uri_case 标志启用不区分大小写的匹配。|否|
|scheme|StringMatch|URI Scheme 值区分大小写。|否|
|method|StringMatch|HTTP 方法区分大小写。|否|
|authority|StringMatch|HTTP 权限值区分大小写。|否|
|headers|map<string,StringMatch>|header中的key必须是小写且使用`-`来分隔，例如 `x-request-id`。|否|
|port|uint32|指定要寻址的主机上的端口。许多服务仅公开单个端口或带有它们支持的协议的标签端口，在这些情况下，不需要明确选择端口。|否|
|sourceLabels|map<string,stirng>|一个或多个标签，用于限制规则对具有给定标签的工作负载的场景。如果 VirtualService 设置了 gateways，则它必须包含mesh以使该字段适用。|否|
|queryParams|map<string,StringMatch>|对query请求的参数进行匹配。|否|
|ignoreUriCase|bool|指定uri匹配中是否区分大小写，默认为否。|否|


## HTTPRouteDestination

每个路由规则都与一个或多个服务版本相关联。与版本相关的权重决定了它接收的流量比例。
例如，以下规则将“评论”服务的 25% 流量路由到带有“v2”标签的实例，其余流量（即 75%）路由到“v1”。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      weight: 75
```

此外，也可以将流量分为两个完全不同的服务，而无需定义新的子集。
例如，以下规则将 25% 的流量转发到 review.com 到 dev.reviews.com 。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route-two-domains
spec:
  hosts:
  - reviews.com
  http:
  - route:
    - destination:
        host: dev.reviews.com
      weight: 25
    - destination:
        host: reviews.com
      weight: 75
```

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|destination|Destination|Destination 唯一标识了请求/连接应该转发到的服务实例。|是|
|weight|int32|要转发到服务版本的流量比例(0-100)。跨目的地的权重总和应该 == 100。如果规则中只有一个目的地，则假定权重值为100。|否|
|headers|Headers|设置Header的操作规则。|否|


## Destination

Destination 表示处理路由规则后请求/连接将发送到的网络可寻址服务。
destination.host 应该明确引用服务注册表中的服务。
Istio 的服务注册表由平台服务注册表中的所有服务（例如 Kubernetes 服务、Consul 服务）以及通过 ServiceEntry 资源声明的服务组成。

Kubernetes 用户注意事项：当使用短名称时（例如“reviews”而不是“reviews.default.svc.cluster.local”），
Istio 将根据规则的命名空间而不是服务来解释短名称。
为避免潜在的错误配置，建议始终使用完全限定域名而不是短名称。

以下 Kubernetes 示例默认将所有流量路由到 Kubernetes 环境中带有标签“version: v1”（即子集 v1）的评论服务的 Pod，以及一些到子集 v2 的流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # interpreted as reviews.foo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v1
```

它对应的 DestinationRule 如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: foo
spec:
  host: reviews # interpreted as reviews.foo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

以下 VirtualService 为 Kubernetes 中对 productpage.prod.svc.cluster.local 服务的所有调用设置了 5 秒的超时。
请注意，此规则中没有定义子集。
Istio 将从服务注册表中获取 productpage.prod.svc.cluster.local 服务的所有实例。
另外需要注意，此规则设置在 istio-system 命名空间中，但使用 productpage 服务的完全限定域名 productpage.prod.svc.cluster.local。
因此，规则的命名空间对解析 productpage 服务的名称没有影响。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-productpage-rule
  namespace: istio-system
spec:
  hosts:
  - productpage.prod.svc.cluster.local # ignores rule namespace
  http:
  - timeout: 5s
    route:
    - destination:
        host: productpage.prod.svc.cluster.local
```

为了控制绑定到网格外部服务的流量的路由，必须首先使用 ServiceEntry 资源将外部服务添加到 Istio 的内部服务注册表。
然后可以定义 VirtualServices 来控制绑定到这些外部服务的流量。
例如，以下规则为 wikipedia.org 定义了一个服务，并为 http 请求设置了 5 秒的超时时间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-wikipedia
spec:
  hosts:
  - wikipedia.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: example-http
    protocol: HTTP
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-wiki-rule
spec:
  hosts:
  - wikipedia.org
  http:
  - timeout: 5s
    route:
    - destination:
        host: wikipedia.org
```

Ps: ServiceEntry 可以将外部服务的访问地址或域名引入Istio中。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|host|string|来自服务注册中心的服务名称。服务名称是从平台的服务注册表（例如 Kubernetes 服务、Consul 服务等）和 ServiceEntry 声明的主机中查找的。如果配置的host在平台服务注册表和ServiceEntry中都没有，则流量会被丢弃。|是|
|subset|string|服务中子集的名称。仅适用于网格内的服务。该子集必须在相应的 DestinationRule 中定义。|否|
|port|PortSelector|指定要寻址的主机上的端口。如果服务仅公开一个端口，则不需要显式选择该端口。|否|


## PortSelector

PortSelector 指定用于匹配或选择最终路由的端口号。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|number|uint32|正确的端口号|否|


## HTTPRedirect




## HTTPRewrite




## Duration




## HTTPRetry




## HTTPFaultInjection




## Headers

当 Envoy 将请求转发到目标服务或从目标服务转发响应时，可以修改消息头。 可以为特定路由目的地或所有目的地指定标头操作规则。
以下 VirtualService 向路由到任何评论服务目标的请求添加了一个值为 test=true 的 headers。
它还删除 foo 响应标头，但仅来自来自评论服务的 v1 子集（版本）的响应。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - headers:
      request:
        set:
          test: true
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      headers:
        response:
          remove:
          - foo
      weight: 75
```

headers 的处理操作可以出现在 HTTPRoute 块，也可以出现在 HTTPRouteDestination 块中。

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|request|HeaderOperations|在将请求转发到目标服务之前对请求headers的操作规则|否|
|response|HeaderOperations|在向调用者返回响应之前要应用的headers操作规则|否|


## HeaderOperations

|字段|类型|字段功能描述|是否必填|
|---|---|----------|-------|
|set|map<string, string>|用给定的map重写header中对应的key|否|
|add|map<string, string>|向headers中添加给定的map信息|否|
|remove|string[]|删除headers中指定的字段|否|

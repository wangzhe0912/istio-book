# Summary

* [Welcome](./README.md)

## 基本概念

* [Istio是什么](chap01/what_is_istio.md)
* [流量管理](chap01/traffic_management.md)
* [安全性](chap01/security.md)
* [可观测性](chap01/observability.md)
* [可扩展性](chap01/extensibility.md)

## 环境搭建

* [快速入门](chap02/quickstart.md)
* [补充说明](chap02/config_profiles.md)

## 示例

* [快速入门](chap04/first_example.md)
* [书店应用](chap04/bookinfo.md)

## 功能介绍

* [流量控制](chap03/traffic_management.md)
  * [请求路由](chap03/request_routing.md)
  * [故障注入](chap03/fault_injection.md)
  * [流量转移](chap03/traffic_shifting.md)
  * [TCP流量转移](chap03/tcp_traffic_shifting.md)
  * [请求超时](chap03/timeout.md)
  * [熔断](chap03/circuit_breaking.md)
  * [镜像流量](chap03/mirroring.md)  
  * [入口流量](chap03/ingress.md)
  * [出口流量](chap03/egress.md)
* [可观测性](chap03/observability.md)
  * [可视化指标](chap03/metrics.md)
  * [日志](chap03/log.md)
  * [分布式追踪](chap03/tracing.md)
  * [可视化网格](chap03/visualizing_mesh.md)

## 运维

* [部署方式](chap05/deployment.md)
  * [架构](chap05/architecture.md)
  * [部署模型](chap05/deployment_models.md)
  * [Pod和Service约束](chap05/requirements.md)
* [最佳实践](chap05/best_practice.md)
  * [部署最佳实践](chap05/deployment_best_practice.md)
* [应用集成](chap05/integration.md)
  * [Grafana](chap05/grafana.md)
  * [Jaeger](chap05/jaegar.md)    
  * [Kiali](chap05/kiali.md)
  * [Prometheus](chap05/prometheus.md)
  * [Zipkin](chap05/zipkin.md)


## 附录

* [配置](chap06/config.md)
  * [流量管理之 VirtualService](chap06/virtual_service.md)
  * [流量管理之 DestinationRule](chap06/destination_rule.md)


## FAQ

* [常见问题](chap07/general.md)
* [Tips](chap07/tips.md)

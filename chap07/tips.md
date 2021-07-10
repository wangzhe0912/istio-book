# Tips

## 修改sidecar镜像地址

在完成了 Istio 环境搭建后，我们可以设置某些 namespace 中可以自动注入 sidecar 容器。
其中，sidecar 镜像的默认镜像地址是: istio/proxyv2:1.8.3

很多时候，我们会在内网中搭建 k8s 集群，其实希望使用私域搭建的镜像仓库，这时可以通过修改 configmap 来设置 sidecar 镜像的仓库地址。

具体来说，设置方式如下：

```shell
kubectl edit configmap istio-sidecar-injector -n istio-system
```

从 configmap 中找出 hub 的配置，并修改为自己的镜像仓库地址即可。

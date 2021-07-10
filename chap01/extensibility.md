# 可扩展性

WebAssembly 是一种沙盒技术，可以用于扩展 Istio 代理（Envoy）的能力。

Proxy-Wasm 沙盒 API 取代了 Mixer 作为 Istio 主要的扩展机制。
在 Istio 1.6 中将会为 Proxy-Wasm 插件提供一种统一的配置 API。

WebAssembly 沙盒的目标：

 - 效率 - 这是一种低延迟，低 CPU 和内存开销的扩展机制。
 - 功能 - 这是一种可以执行策略，收集遥测数据和执行有效荷载变更的扩展机制。
 - 隔离 - 一个插件中程序的错误或是崩溃不会影响其它插件。
 - 配置 - 插件使用与其它 Istio API 一致的 API 进行配置。可以动态的配置扩展。
 - 运维 - 扩展可以以仅日志，故障打开或者故障关闭的方式进行访问和部署。
 - 扩展开发者 - 可以用多种编程语言编写。

关于  WebAssembly 集成架构的介绍可以参考 [视频](https://youtu.be/XdWmm_mtVXI) 。

## 高级架构



## 示例



## SDK



## 生态




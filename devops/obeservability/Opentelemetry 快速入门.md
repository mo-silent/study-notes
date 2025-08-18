# Opentelemetry 是什么

OpenTelemetry是一个观测性框架和工具包，旨在创建和管理遥测数据，如追踪、指标和日志。OpenTelemetry是厂商和工具无关的，意味着它可以与各种观测性后端一起使用，包括像Jaeger和Prometheus这样的开源工具，以及商业解决方案。OpenTelemetry是一个Cloud Native Computing Foundation（CNCF）项目。

Opentelemetry 包含下面主要组件：

- 用于所有组件的规范。
- 定义遥测数据形状的标准协议。
- 为常见遥测数据类型定义标准命名方案的语义约定。
- 定义如何生成遥测数据的API。
- 实现规范、API和遥测数据导出的语言SDK。
- 实现常见库和框架的仪表化的库生态系统。
- 无需更改代码生成遥测数据的自动仪表化组件。
- OpenTelemetry Collector：接收、处理和导出遥测数据的代理。
- 其他各种工具，例如OpenTelemetry Operator for Kubernetes、OpenTelemetry Helm Charts和用于FaaS的社区资产。
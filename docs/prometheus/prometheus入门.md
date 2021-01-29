
## What is Prometheus Monitoring? 😍

Prometheus是一款时序（time series）数据库；但它的功能却并非止步于TSDB，而是一款设计用于进行目标（Target）监控的关键组件；

结合生态系统内的其它组件，例如pushgateway、altermanager和grafana等，可构成一个完整的IT监控系统；
![prometheus-grafana](img/wx-20210128.png)

## 时序数据简介

时序数据，是在一段时间内通过重复测量（measurement）而获得的观测值的集合；将这些观测值绘制于图形之上，它会有一个数据轴和时间轴；

服务器指标数据、应用程序性能监控数据、网络数据等，也都是时序数据；
![shixu](img/shixushuju.png)

## What does Prometheus do?

Prometheus基于HTTP call，从配置文件中指定的网络端点（endpoint）上周期性获取指标数据

- Exporters
- Instrumentation
- Pushgateway

![httpcall](img/httpcall.png)

## Pull and Push

Prometheus同其它TSDB相比又一个非常典型的特效：它主动从各Target上"`拉取（pull）`"数据,而非等待被监控端的"`推送（push）`"；

两个方式各有优劣，其中pull模型的优势在于:
- 集中控制：有利于将配置集在Prometheus Server上完成，包括指标及才去速率等；
- Prometheus的根本目标在于收集Target上预先完成聚合的聚合型数据，而非一款由事件驱动的存储系统；
![pull](img/pull.png)

## Prometheus的生态组件

Prometheus生态圈中包含了多个组件，其中部分组件可选

- Prometheus Server：收集和存储事件序列和数据；
- Client Library： 客户端库，目的在于为那些期望原生提供`Instrumentation`功能的应用程序提供便捷的开发途径；
- Push Gateway： 接收那些通常由短期作业生成的指标数据的网关，并支持有Prometheus Server进行指标拉取操作；
- Exporters：用于暴露现有应用程序或服务(不支持Instrumentation)的指标给Prometheus Server；
- Alertmanager：从Prometheus Server接收到`告警通知`后，通过去重、分组、路由等预处理功能后以高效向用户完成告警信息发送；
- Data Visualization：Prometheus Web UI(Prometheus Server内建)，及`Grafana`等；
- Service Discovery：动态发现待监控的Target，从而完成监控配置的重要组件，在容器化环境中尤为有用；该组件目前由Prometheus Server内建支持；
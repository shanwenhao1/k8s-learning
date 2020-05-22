# 集群监控工具Prometheus

Prometheus是Google内部监控报警系统的开源版本, 是Google [SRE](https://baike.baidu.com/item/SRE/1141123?fr=aladdin)
思想在其内部不断完善的产物, 它的存在是为了更快和高效的发现问题、快速的接入速度.

## prometheus介绍

### 特点
- 具有由 metric 名称和键/值对标识的时间序列数据的多维数据模型
- 有一个灵活的查询语言
- 不依赖分布式存储，只和本地磁盘有关
- 通过 HTTP 的服务拉取时间序列数据
- 也支持推送的方式来添加时间序列数据
- 还支持通过服务发现或静态配置发现目标
- 多种图形和仪表板支持

###组件构成
- Prometheus Server: 用于抓取指标、存储时间序列数据
- exporter:暴露指标让任务来抓
- pushgateway: push 的方式将指标数据推送到该网关
- alertmanager: 处理报警的报警组件
- adhoc: 用于数据查询

### 架构
![](../../../../doc/picture/k8s/prometheus/prometheus-architecture.png)


## [安装](prometheus%20build.md)

## [使用教程](probetheus%20use/prometheus%20use.md)

## [Prometheus报警配置](alertManager/alertManager.md)
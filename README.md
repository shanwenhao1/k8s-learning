# 日志收集系统

[Flume](https://flume.apache.org/) + [elastic.co](https://www.elastic.co/) + [Kibana](https://www.elastic.co/products/kibana)


~~p[Elasticsearch权威指南(中文版)](https://www.gitbook.com/book/looly/elasticsearch-the-definitive-guide-cn/details)p~~

# kwFlume

Flume是分布式、可靠且可用的有效收集、聚合、大批量移动日志的服务. 其拥有简单、灵活的基于流数据的架构, 它通过故障转移
和恢复机制来确保可靠性. 它使用简单的可扩展的数据模型, 以允许在线分析应用程序. 

在安装Flume之前, 建议先部署kubernetes 集群, [部署方案](#K8s部署)

## Prepare
- [Flume准备](doc/flume/prepare.md)
- [elastic search准备](doc/elastic%20search/prepare.md)
- [kibana准备](doc/kibana/prepare.md)

## Install
- [flume install](doc/flume/install.md)
- [elastic install](doc/elastic%20search/install.md)
- [kibana install](doc/kibana/install.md)

## K8s部署
[Docker相关](doc/docker/docker.md)

[k8s部署](doc/k8s/k8s%20deployment.md)


## Reference material
[dokcer-k8s-elk一站式](https://www.qikqiak.com/k8s-book/)

### official document
- [Elasticsearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

### other blog
- [实时日志收集-查询-分析系统(Flume+ElasticSearch+Kibana)](https://yq.aliyun.com/articles/601055)
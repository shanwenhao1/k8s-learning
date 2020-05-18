# Heapster插件

Heapster是容器集群监控和性能分析工具, 天然支持kubernetes.

[官方地址](https://github.com/kubernetes-retired/heapster)

## 部署Heapster插件

- 下载yaml部署文件
    - 从官网手动下载Heapster.yaml文件: [influxdb.yaml](https://github.com/kubernetes-retired/heapster/tree/master/deploy/kube-config/influxdb/influxdb.yaml)、
    [heapster-rabc.yaml](https://github.com/kubernetes-retired/heapster/blob/master/deploy/kube-config/rbac/heapster-rbac.yaml)
        - 此种方法需要手动更改镜像源
    - 命令行方式下载(镜像源已更改为国内可用)
        ```bash
        wget http://mirror.faasx.com/kubernetes/heapster/deploy/kube-config/influxdb/influxdb.yaml
        
        ```
- kubectl执行部署
    ```bash
    kubectl create -f influxdb.yaml
    ```
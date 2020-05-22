# 日志收集 

## 日志收集方案

kubernetes集群本身不提供日志收集的解决方案, 一般来说主要有三种方案来做日志收集:
- 在节点上运行一个agent来收集日志
    - 必须在每个节点上运行一个代理程序, 所以直接使用DaemonSet控制器运行该应用程序即可
    - 对应用程序没有任何侵入性.
    - 缺点是该方法仅适合收集输出到stdout和stderr的应用程序日志
    ![](../../../picture/k8s/log/logging-with-node-agent.png)
- 在Pod中包含一个sidecar容器来收集应用日志
    - 用于采集应用程序输出到容器中日志文件的日志信息
        - 方法是, 在Pod中启动另外一个sidecar容器, 将应用程序的日志通过这个容器重新输出到stdout.
        由于重定向输出到了stdout或者stderr, 所以我们可以使用`kubectl logs`来查看日志.
        - 缺点是不仅在原容器文件中占用空间, 也会通过stdout输出后占用磁盘空间
    ![](../../../picture/k8s/log/logging-with-streaming-sidecar.png)
- 直接在应用程序中将日志信息推送到采集后端
    - 创建一个单独的日志采集代理程序的sidecar容器, 需要单独配置和应用程序一起运行, 比较灵活. 
    - 缺点是会造成大量的资源消耗(因为有多少个需要采集的pod就需要运行多少个采集代理程序)
    ![](../../../picture/k8s/log/logging-with-sidecar-agent.png)
- 直接从应用程序收集日志: 从代码层面上在应用程序中将日志推送至日志后端
    ![](../../../picture/k8s/log/logging-from-application.png)


## EFK

Kubernetes中比较流行的日志收集解决方案是Elasticsearch、Fluentd、Kibana(EFK)技术栈.
- [Elasticsearch](../../../elastic%20search/prepare.md)详情
- [Fluentd](../../../fluentd/prepare.md)详情
- [Kibana](../../../kibana/prepare.md)详情


## EFK搭建(Helm)
事先准备: 添加需要的helm Charts包源
```bash
# 添加elastic官方源
helm repo add elastic https://helm.elastic.co
# 添加k8s官方源
helm repo add stable https://kubernetes-charts-incubator.storage.googleapis.com/
```


### Elasticsearch官方部署:
部署所需配置文件都在[helm elastic](helm%20elastic)目录下
- 添加elastic官方源
    ```bash
        # 添加官方源
        helm repo add elastic https://helm.elastic.co
    ```
- 搭建elasticsearch集群, [官方文档](https://github.com/elastic/helm-charts/tree/master/elasticsearch)
官方配置文件[es values.yaml](helm%20elastic/es%20values.yaml), 按需修改后的虚拟环境下的配置文件
[es values change.yaml](helm%20elastic/es%20values%20change.yaml)
    - 由于测试部署中使用了storageClass持久化数据, 
    因此首先需要创建[es-storageclass.yaml](helm%20elastic/es-storageclass.yaml)
    ```bash
    # 创建storage class
    kubectl apply -f es-storageclass.yaml
    # 部署elasticsearch集群, 其中es-data为上一步创建的storage class名称
    helm install --name elasticsearch elastic/elasticsearch -f es_values_change.yaml
    ```
- 部署kibana, [官方文档](https://github.com/elastic/helm-charts/tree/master/kibana)
官方配置文件[kibana values.yaml](helm%20elastic/kibana%20values.yaml), 
修改后的文件[kibana values change.yaml](helm%20elastic/kibana%20values%20change.yaml)
    ```bash
    helm install --name kibana elastic/kibana -f kibana_values_change.yaml
    ```
- 安装fluentd-elasticsearch, [文档](https://hub.helm.sh/charts/kiwigrid/fluentd-elasticsearch),
    - k8s [stable/fluentd文档](https://github.com/helm/charts/tree/master/stable/fluentd-elasticsearch)
    部署文件[fluentd-elasticsearch values.yaml](helm%20elastic/fluentd-elasticsearch%20values.yaml),
    修改后的部署文件[fluentd-elasticsearch values change.yaml](helm%20elastic/fluentd-elasticsearch%20values%20change.yaml)
    ```bash
    helm install --name fluentd-elasticsearch stable/fluentd-elasticsearch -f fluentd-elasticsearch_values_change.yaml
    ```
    







### Kubernetes官方部署:
- 创建storage class以供elasticsearch挂载, [es-storageclass.yaml](helm/es-storageclass.yaml)
- [elasticsearch 文档](https://github.com/helm/charts/tree/master/stable/elasticsearch),
[helm hub of elasticsearch](https://hub.helm.sh/charts/stable/elasticsearch)
根据自己的需求更改默认的[es values.yaml](helm/es%20values.yaml), 
修改后的文件为[es values change.yaml](helm/es%20values%20change.yaml)
    - install elasticsearch
        ```bash
        # 先创建storageClass
        kubectl apply -f es-storageclass.yaml
        # helm部署elasticsearch, 生产环境中存储空间可使用较大一点的配置
        # 命名最好不要更改, 如若更改则需要更改fluent及kibana连接地址, 否则无法连接到elasticsearch
        helm install stable/elasticsearch --name elasticsearch -f es_values_change.yaml
        ```
        - 可用以下命令查看部署情况
            ```bash
            # kubectl port-forward --namespace default $POD_NAME 9200:9200
            kubectl port-forward --namespace default my-elasticsearch-master-0 9200:9200
            curl http://localhost:9200/_cluster/state?pretty
            ```
        - 撤销部署
        ```bash
        helm del --purge my-elasticsearch
        # kubectl delete pvc -l release=my-elasticsearch
        kubectl delete pvc -l release=elasticsearch,component=data
        ```
- [Fluentd Helm Chart](https://hub.helm.sh/charts/stable/fluentd)
    - [github of fluentd](https://github.com/kiwigrid/helm-charts/tree/master/charts/fluentd-elasticsearch),
    根据自己的需求更改默认的[fluentd values.yaml](helm/fluentd%20values.yaml)   
    - install fluentd
        ```bash
        # 查找fluentd chart
        helm search stable/fluentd
        # 查看stable/fluentd的信息
        helm inspect stable/fluentd
        # 安装, 也可根据自己配置的文件方式安装 helm install stable/fluentd --name my-release -f values.yaml
        helm install stable/fluentd --name fluentd
        # 或者
        helm install stable/fluentd --name my-release -f values.yaml
        # 卸载命令 helm delete my-fluentd
        # 验证安装是否成功
        kubectl --namespace=default get all -l "app=fluentd,release=my-fluentd"
        ```
- [Kibana Helm Chart](https://hub.helm.sh/charts/elastic/kibana)
    - [github of kibana](https://github.com/helm/charts/tree/master/stable/kibana)
    根据自己的需求更改默认的[kibana values.yaml](helm/kibana%20values.yaml)
    修改后的文件为[kibana values change.yaml](helm/kibana%20values%20change.yaml)
    - install kibana
        ```bash
        # 记得将kibana_values.yaml中的elasticsearch.hosts改成部署的elasticsearch的url
        helm install stable/kibana --name kibana -f kibana_values_change.yaml
        ```


## EFK搭建(Yaml)
[搭建参考](efk.md#部署efk)


## 遇到的问题
- [elasticsearch cluster failing by connection refused to port 9200 and discovery fails](https://github.com/helm/charts/issues/10791)
    
## 参考
- [kubernetes部署方案](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)
    [kubernetes efk yaml文件](https://github.com/kubernetes/kubernetes/tree/release-1.16/cluster/addons/fluentd-elasticsearch)
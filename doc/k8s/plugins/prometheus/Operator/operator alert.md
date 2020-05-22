# Operator Alert

@TOC
- [添加自定义Operator监控项](#添加自定义Operator监控项)
    - [添加etcd监控](#添加etcd监控)
    - [自定义报警](#自定义报警)
        - [添加PrometheusRule](#添加PrometheusRule)
        - [配置报警](#配置报警)
- [自动发现报警配置](#自动发现报警配置)

## 添加自定义Operator监控项

添加自定义监控的大致流程:
- 建立一个ServiceMonitor对象, 用于Prometheus添加监控项
- 为ServiceMonitor对象关联metrics数据接口的一个Service对象
- 确保Service对象可以正确获取到metrics数据


### 添加etcd监控

- 获取etcd证书, 用以访问etcd服务
    ```bash
    kubectl get pod etcd-ubuntu-master-one -n kube-system -o yaml
    ```
    ![](../../../../picture/k8s/operator/operator%20etcd%20ca1.png)
    可见etcd证书在`/etc/kubernetes/pki/etcd/`目录下
- 将需要的证书通过secret对象保存到集群中去
    ```bash
    kubectl create secret generic etcd-certs -n monitoring --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key --from-file=/etc/kubernetes/pki/etcd/ca.crt
    ```
- 将上面创建的`etcd-certs`对象配置到prometheus资源对象中
    - 直接编辑
        ```bash
        kubectl edit prometheus k8s -n monitoring
        ```
    - 或者, 更改yaml文件[prometheus-prometheus.yaml](manifests/prometheus-prometheus.yaml)
        ```yaml
        nodeSelector:
          beta.kubernetes.io/os: linux
        replicas: 2
        # 添加的部分
        secrets:
        - etcd-certs
        ```
- 创建etcd的ServiceMonitor对象[prometheus-serviceMonitorEtcd.yaml](etcd%20alert/prometheus-serviceMonitorEtcd.yaml)
- 创建相应的Service对象[prometheus-etcdService.yaml](etcd%20alert/prometheus-etcdService.yaml)
    ![](../../../../picture/k8s/operator/operator%20alert%20etcd.png)
- 在grafana webUI中导入`3070`的dashboard
    ![](../../../../picture/k8s/operator/operator%20alert%20dashboard.png)
 
 
### 自定义报警
  
#### 添加PrometheusRule
由Pormetheus dashboard的Config我们可以看到: alertmanagers实例的配置是通过角色为endpoints的
kubernetes的服务发现机制获取的. 匹配的是服务名为alertmanager-main、Port名为web. 
![](../../../../picture/k8s/operator/alert%20conf%20role.png)
相应的报警规则文件位于`/etc/prometheus/rules/prometheus-k8s-rulefiles-0/`所有的Yaml文件.
![](../../../../picture/k8s/operator/prometheus%20rule.png)
这个Yaml实际上是我们之前创建[manifests](manifests)目录下的[prometheus-rules.yaml](manifests/prometheus-rules.yaml)

**[prometheus-prometheus.yaml](manifests/prometheus-prometheus.yaml)中的ruleSelector用来匹配rule过滤规则**

所以我们要自定义一个报警规则, 只需要创建一个具有`prometheus=k8s`和`role=alert-rules`的PrometheusRule对象即可


以ectd为例(不可用的etcd数量超过了一半就触发报警), 创建[prometheus-etcdRules.yaml](etcd%20alert/prometheus-etcdRules.yaml)
![](../../../../picture/k8s/operator/prometheus%20rule2.png)
上图可见, prometheus的pod下面已经加载了etcd的报警规则

在prometheus的dashboard中可见我们自定义的etcd报警已经生效
![](../../../../picture/k8s/operator/etcd%20alert%20success.png)


#### 配置报警
有了报警规则, 接下来配置报警信息发送方式.
- 修改alertManager的service [alertmanager-service.yaml](manifests/alertmanager-service.yaml), 改为nodePort类型
- 将[alertmanager-secret.yaml](manifests/alertmanager-secret.yaml)中的alertmanager.yaml对应的value做base64解码
    ```bash
    echo Imdsb2JhbCI6CiAgInJlc29sdmVfdGltZW91dCI6ICI1bSIKInJlY2VpdmVycyI6Ci0gIm5hbWUiOiAibnVsbCIKInJvdXRlIjoKICAiZ3JvdXBfYnkiOgogIC0gImpvYiIKICAiZ3JvdXBfaW50ZXJ2YWwiOiAiNW0iCiAgImdyb3VwX3dhaXQiOiAiMzBzIgogICJyZWNlaXZlciI6ICJudWxsIgogICJyZXBlYXRfaW50ZXJ2YWwiOiAiMTJoIgogICJyb3V0ZXMiOgogIC0gIm1hdGNoIjoKICAgICAgImFsZXJ0bmFtZSI6ICJXYXRjaGRvZyIKICAgICJyZWNlaXZlciI6ICJudWxsIg== | base64 -d
    ```
    - 解码后的`alertmanager.yaml`
        ```yaml
        "global":
          "resolve_timeout": "5m"
        "receivers":
        - "name": "null"
        "route":
          "group_by":
          - "job"
          "group_interval": "5m"
          "group_wait": "30s"
          "receiver": "null"
          "repeat_interval": "12h"
          "routes":
          - "match":
              "alertname": "Watchdog"
            "receiver": "null"
        ```
    - 创建文件[alertmanager.yaml](etcd%20alert/alertmanager.yaml), 添加我们想要的接收器
        ```bash
        kubectl delete secret alertmanager-main -n monitoring
        kubectl create secret generic alertmanager-main --from-file=alertmanager.yaml -n monitoring
        ```
        至此, 我们自定义Prometheus报警已经完成了
        
        
        
## 自动发现报警配置

随着集群中的Service/Pod越来越多, 我们如果需要一个个去建立相应的ServiceMonitor对象, 是不是太麻烦了.

因此Prometheus Operator为我们提供了一个额外的抓取配置来解决这个问题, 通过这个添加的额外配置来进行服务发现的自动监控

接下来我们以Operator去自动监控具有`prometheus.io/scrape=true`这个annotations的Service.
还记得[prometheus部署(学习)](../prometheus.md)中[prometheus-cm.yaml](../prometheus-cm.yaml)文件中的
```yaml
      # 自动发现集群中的Service
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        # 配置为true时, 想要自动发现集群中的service则需要在annotation区域添加`prometheus.io/scrape=true`的声明
        regex: true
```
配置嘛, 接下来我们将`kubernetes-service-endpoints`这个监控配置copy下来写入到创建的
[prometheus-additional.yaml](etcd%20alert/prometheus-additional.yaml)中
- 接着创建对应的secret对象
    - 两种创建方法:
        - 手动创建
            ```bash
            kubectl create secret generic additional-configs --from-file=prometheus-additional.yaml -n monitoring
            ```
            - 创建完成后会将上面的配置信息进行base64编码后作为`prometheus-additional.yaml`这个key对应的存在
            ![](../../../../picture/k8s/operator/additional%20secret%20base64.png)
        - 通过[prometheus-additional-secret.yaml](manifests/prometheus-additional-secret.yaml)文件创建
- 在声明prometheus的资源对象文件[prometheus-prometheus.yaml](manifests/prometheus-prometheus.yaml)中
添加这个额外配置
    ```yaml
      # 添加的alert自动发现配置, 详情请见本文件上级目录中的operator alert.md
      additionalScrapeConfigs:
        name: additional-configs
        key: prometheus-additional.yaml
    ```
    并生效
    ```bash
    kubectl apply -f prometheus-prometheus.yaml
    ```
    可以看到, 我们的service endpoints监控已经起来了
    ![](../../../../picture/k8s/operator/service%20endpoints.png)
    接下来我们只需要在我们的服务下添加具有`prometheus.io/scrape=true`的annotations就可以实现自动发现监控了.
    使用请参考[redis](../probetheus%20use/prome-redis.yaml)
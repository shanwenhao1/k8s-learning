# Grafana 安装使用

Grafana是一个可视化面板、有着非常漂亮的图表和布局展示. 支持Graphite、zabbix、influxDB、Prometheus、OpenTSDB、
ElasticSearch等作为数据源.

## 部署

- 提前创建pvc持久化数据, [grafana-volume.yaml](grafana-volume.yaml)
    ```bash
    kubectl create -f grafana-volume.yaml
    # 查看pvc是否创建成功
    kubectl get pvc -n kube-ops
    ```
- 部署pod, [grafana-deploy.yaml](grafana-deploy.yaml)
    ```bash
    kubectl apply -f grafana-deploy.yaml
    # 查看部署情况
    kubectl get pod -n kube-ops
    ```
- 由于5,1版本后groupdid更改了导致`/var/lib/grafana`目录的权限问题, 目录挂载到pvc目录后, 
拥有者并不是`grafana-deploy.yaml`, 执行一个job任务更改该目录所属用户.
[grafana-chown-job.yaml](grafana-chown-job.yaml)
    ```bash
    kubectl create -f grafana-chown-job.yaml
    ```
- 通过service对外暴露服务
    ```bash
    kubectl create -f grafana-svc.yaml
    ```
    ![](../../../picture/k8s/grafana/grafana%20build%20complete.png)
- 通过任意节点上的`32356`端口即可访问grafana的web了, 通过deployment中配置的管理员账户、
密码进行登录访问
    ![](../../../picture/k8s/grafana/grafana%20web%20start.png)
    - 点击`Add data source`添加数据源
        - 添加`Prometheus`(内网模式)
            ![](../../../picture/k8s/grafana/add%20data%20sources%20prometheus%20proxy%20netword.png)
        - 添加`Prometheus`(外网模式)
            ![](../../../picture/k8s/grafana/add%20data%20sources%20prometheus.png)
    - 返回主页, 点击左侧栏`+`  `Create`    `Import` 
        ![](../../../picture/k8s/grafana/add%20create%20dashboard.png)
        - 选择[Kubernetes cluster monitoring (via Prometheus)(dashboard id 为162)](https://grafana.com/grafana/dashboards/162/revisions)
        作为kubernetes集群展示的dashboard.
        (你也可以选择自己喜欢的dashboard导入[选择](https://grafana.com/grafana/dashboards))
            ![](../../../picture/k8s/grafana/add%20create%20dashboard%20import.png)
            ![](../../../picture/k8s/grafana/add%20create%20dashboard%20import2.png)
    - 由于dashboard所需要的数据指标名称和我们Prometheus采集的数据指标不一致, 会造成无法正确获取数据, 
    如图
        ![](../../../picture/k8s/grafana/need%20change.png)
    需要我们做如下修改:
        - 修改`cluster memory usage`
            ```bash
            # (整个集群的内存-(整个集群剩余的内存以及Buffer和Cached))/整个集群的内存
            (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100
            ```
        ![](../../../picture/k8s/grafana/cluster%20memory%20usage.png)
        - 修改`cluster CPU usage`
            ```bash
            sum(sum by (pod_name)(rate(container_cpu_usage_seconds_total{image!="", pod_name!=""}[1m] ) )) / count(node_cpu_seconds_total{mode="system"}) * 100
            ```
        - 修改`cluster Filesystem usage`
            ```bash
            # (整个集群的存储空间-集群剩余的存储空间)/整个集群的存储空间
            (sum(node_filesystem_size_bytes{device="/dev/sda1"}) - sum(node_filesystem_free_bytes{device="/dev/sda1"}) ) / sum(node_filesystem_size_bytes{device="/dev/sda1"}) * 100
            ```
                   
## 插件

### kubernetes插件
grafana 有为kubernetes集群监控提供一个插件, [grafana-kubernetes-app](https://grafana.com/grafana/plugins/grafana-kubernetes-app/installation)

安装
- 要安装这个插件, 需要到grafana的Pod里面去执行安装命令
    ```bash
    kubectl get pods -n kube-ops
    # 进入grafana容器中
    kubectl exec -it grafana-6c47858d55-jg4q7 /bin/bash -n kube-ops
    # 在容器中执行命令安装插件
    bin/grafana-cli plugins install grafana-kubernetes-app
    # 离开容器
    exit
    ```
    ![](../../../picture/k8s/grafana/install%20plugin%20kubernetes.png)
- 安装插件后需要重启grafana, 这里采用删除pod(k8s会去重新启动一个pod)
    ```bash
    kubectl delete pods grafana-6c47858d55-jg4q7 -n kube-ops
    ```
    至此, kubernetes插件已经在grafana下安装完成了, 可以在`setting/plugins`下找到他
- 配置kubernetes
    - 创建在`/root/.kube/ca`目录, 并在目录下创建`admin-ca.txt`、`admin-client-key.txt`、`admin-client-certificate.txt`
    存入`/etc/kubernetes/admin.conf`中对应的`certificate-authority-data`、`client-key-data`、`client-certificate-data`
    - 使用命令对这三个txt文件进行解码并存入相应文件
        ```bash
        cat admin-ca.txt | base64 -d > admin-ca.ca
        cat admin-client-key.txt | base64 -d > admin-client.key
        cat admin-client-certificate.txt | base64 -d > admin-client.crt
        # 可用以下命令查看生成的认证文件信息
        openssl x509 -in ./admin-client.crt -text
        ```
    - 最后, 将相应文件的tls认证信息填到下图中的对应位置
        ![](../../../picture/k8s/grafana/kubernetes%20config1.png)
        - prometheus 数据源勾选我们之前配置的[prometheus](../prometheus/prometheus.md)
        - 这样我们的kubernetes插件就安装完成了, 使用:
        ![](../../../picture/k8s/grafana/use%20kubernetes1.png)
        ![](../../../picture/k8s/grafana/use%20kubernetes2.png)
        ![](../../../picture/k8s/grafana/use%20kubernetes3.png)
    
    
    
## 报警

grafana支持多种形式的报警功能, email、钉钉等. 这里我们写下一些示例. 不过由于grafana自带的报警功能比较弱. 
因此生产环境中一般不使用grafana自带的报警功能. **而是使用[AlertManager](../prometheus/alertManager/alertManager.md)**

### email报警
- 使用邮箱开启smtp转发服务, 并将smtp密钥拷贝备用
![](../../../picture/k8s/grafana/alert%20mail%20config.png)
- 启用email报警需要在启动配置文件中`etc/grafana/grafana.ini`中开启SMTP服务, 这里我们使用configmap
资源对象挂载到grafana 中, [grafana-cm.yaml](grafana-cm.yaml)
    ```bash
    kubectl create -f grafana-cm.yaml
    # 将smtp密钥置入密码处
    ```
- 修改[grafana-deploy.yaml](grafana-deploy.yaml), 将configmap挂载入pod
    - 更改deploy文件
        ```yaml
          volumeMounts:
          - mountPath: "/etc/grafana"
            name: config
        volumes:
        - name: config
          configMap:
            name: grafana-config
        ```
    - 更行pod
        ```bash
        kubectl apply -f grafana-deploy.yaml
        # 如果apply更新pod在后续测试中未能成功报警, 请检查邮箱配置(configmap)或删掉grafana pod重新部署
        # kubectl delete -n monitoring pods grafana-c7c67db54-wvbdx -n kube-ops
        ```
- 测试报警, 在grafana的webui中`Alert`页面测试
    ![](../../../picture/k8s/grafana/alert%20email%20test.png)
    
    
### 钉钉报警

- 在钉钉群聊中添加`自定义机器人`, 在电脑端钉钉中复制机器人管理中的`webhook`链接
- 在grafana webui中将链接配置
    ![](../../../picture/k8s/grafana/alert%20dingding%20config.png)
    
### 使用UI自定义报警

- 创建自定义alert报警规则
    ![](../../../picture/k8s/grafana/graph%20alert.png)
- alert规则配置及说明
    ![](../../../picture/k8s/grafana/alert%20rule.png)
    - 1: Alert名称
    - 2: 执行频率
    - 3: 判断标准, 默认是avg(下拉框可自由选择)
    - 4: query(A,5m,now), 字母A代表选择的metrics中设置的sql, 也可以选择其它在metrics中设置的, 但这里是单选. 
    5m代表从现在起往之前的五分钟, 即5m之前的那个点为时间的起始点, now为时间的结束点, 此外这里可以自己手动输入时间.
    - 5: 设置的预警临界点, 手动输入
    - 6: 与5功能相同, 手动移动
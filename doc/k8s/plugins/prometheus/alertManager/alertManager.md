# AlertManager

alertManager是Prometheus的一个报警模块, 支持丰富的告警通知渠道, 很容易对告警信息进行去重, 降噪, 分组等.

回顾下我们之前学到的Prometheus架构
![](../../../../picture/k8s/prometheus/prometheus-architecture.png)

## 安装

- 更改[prometheus-cm.yaml](../prometheus-cm.yaml), 添加alterManager的configMap
    ```bash
    kubectl create -f prometheus-cm.yaml
    ```
- 在之前的Prometheus部署文件[prometheus-deploy.yaml](../prometheus-deploy.yaml)中添加以下配置
    ```yaml
      - name: alertmanager
        image: prom/alertmanager
        imagePullPolicy: IfNotPresent
        args:
        - "--config.file=/etc/alertmanager/config.yml"
        - "-storage.path=/alertmanager"
        ports:
        - containerPort: 9093
          name: http
        volumeMounts:
        - mountPath: "/etc/alertmanager"
          name: alertcfg
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi
    volumes:
    - name: alertcfg
      configMap:
        name: alert-config
    ```
    - 更新Prometheus这个pod
        ```bash
        kubectl apply -f prometheus-deploy.yaml
        ```
- 更改prometheus configMap部署文件[prometheus-cm.yaml](../prometheus-cm.yaml)
    - 在[prometheus-cm.yaml](../prometheus-cm.yaml)中添加以下配置, 使得Prometheus能够访问到alertManager
        ```yaml
        alerting:
          alertmanagers:
            - static_configs:
              - targets: ["localhost:9093"]
        ```
    - 添加报警规则配置
       ```yaml
        rule_files:
        - /etc/prometheus/rules.yml
       ```
    - 更新`prometheus-cm.yaml`
        ```bash
        kubectl delete -f prometheus-cm.yaml
        kubectl create -f prometheus-cm.yaml
        # 刷新配置使得配置生效
        curl -X POST "http://10.108.209.234:9090/-/reload"
        ```
        ![](../../../../picture/k8s/alert%20manager/build.png)
- 在prometheus的web UI查看alert情况
![](../../../../picture/k8s/alert%20manager/alert%20test.png)
    - 报警信息有以下三种状态
        - inactive((0 active)): 当前无报警
        - pending: 表示在设置的阈值时间范围内被激活了
        - firing: 表示超过设置的阈值时间范围被激活了
    - 并且我们配置的报警接收邮箱会收到报警邮件
        ![](../../../../picture/k8s/alert%20manager/alert%20mail.png)
    - 可以看到, 在邮件中有`Source`的链接, 因此我们可以通过nodePort的方式去查看AlertManager的Dashboard.
       - 首先在[prometheus-svc.yaml](../prometheus-svc.yaml)添加alertManger的nodePort配置
           ```bash
           # alertManger
           - name: alert
             port: 9093
           ```
       - 更新svc
            ```bash
            kubectl apply -f prometheus-svc.yaml
            # 获取nodePort暴露的端口
            kubectl get svc -n kube-ops
            # 通过任意节点+端口访问
            # http://k8s.swh.com:31107/#/alerts
            ```
            ![](../../../../picture/k8s/alert%20manager/alert%20dashboard.png)
            **在alertManager**的dashboard界面我们可以设置一些报警信息的过滤、slience等操作
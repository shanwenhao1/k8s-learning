# k8s Deployment

## [k8s的一些概念](../../doc/k8s/k8s%20base%20document.md)

## k8s集群搭建
[快速搭建(推荐)](../../doc/k8s/k8s%20cluster%20build.md)

### k8s必要插件
- [k8s集群管理UI: dashboard](plugins/dashboard/dashboard.md)
- [外部服务暴露工具: Ingress](plugins/ingress/ingress.md)
- [包管理工具Helm](plugins/Helm/helm.md)
- [分布式存储系统Ceph](plugins/ceph/ceph.md)
- Kubernetes集群监控:
    - 生产环境中推荐使用`Operator`部署方式: 
        - [自动化部署kubernetes集群监控-Operator](plugins/prometheus/Operator/operator.md)
        - [自定义Prometheus Operator监控](plugins/prometheus/Operator/operator%20alert.md)
    - 部署教程式部署方式:
        - ~~1[k8s早期集群监控和性能分析工具Heapster](plugins/Heapster/Heapster.md)(不使用)1~~
        - [prometheus](plugins/prometheus/prometheus.md)
            - [prometheus报警工具alertManager](plugins/prometheus/alertManager/alertManager.md)
        - [可视化工具Grafana](plugins/grafana/Grafana.md)
- [日志收集架构](plugins/EFK/efk2.md)



## 其他资料
- [k8s安装部署摘要](./k8s集群部署操作手册.pdf)
- [dokcer-k8s-elk一站式](https://www.qikqiak.com/k8s-book/)
- [kubernetes指南](https://feisky.gitbooks.io/kubernetes/)
    

## 常用命令

```bash
# 查看集群node数量
kubectl get nodes
# 获取node详细信息
kubectl describe node <node_name>
# 查看所有pod运行状态
kubectl get pods --all-namespaces
kubectl get pod --all-namespaces -o wide
# 获取pod的ip及暴露的端口信息
kubectl get ep -n kube-system
# 获取pod详细信息
kubectl describe pod your_pod_name -n your_namespace
kubectl describe pod coredns-fb8b8dccf-ngsh5 -n kube-system
# 获取pod日志
kubectl logs podName -n kube-system
# 删除pod
kubectl delete pod pod_name
# 创建token(默认过期时间是24h)
kubeadm token generate
kubeadm token create

# 删除某个节点, --ignore-daemonsets为master节点使用
kubectl drain ubuntu-node-1 --delete-local-data --ignore-daemonsets
kubectl delete node ubuntu-node-1
kubect get nodes

# 查看service信息
kubectl get svc kubernetes -o yaml

# 删除所有已退出的docker容器    谨慎使用
docker rm `docker ps -a|grep Exited|awk '{print $1}'`
# 配置外网访问
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'
# 强制删除pod
kubectl delete pod --grace-period=0 --force node-exporter-srzkk -n monitoring
# 启动一个容器用以测试与其他容器的一些操作
kubectl run  -it --rm  cirror-$RANDOM --image=cirros -- /bin/sh
```


docker相关命令
```bash
# 删除tag为none的镜像elasticsearch-logging:9200
docker images|grep none|awk '{print $3}'|xargs docker rmi
# 批量删除状态为exited的容器
docker rm $(docker ps -q -f status=exited)
```
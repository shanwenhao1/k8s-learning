# k8s Deployment

## [k8s的一些概念](../../doc/k8s/k8s%20base%20document.md)

## k8s集群搭建
[快速搭建(推荐)](../../doc/k8s/k8s%20cluster%20build.md)

### k8s必要插件
- [k8s集群管理UI: dashboard](plugins/dashboard/dashboard.md)
- Kubernetes集群监控:
    - ~~1[k8s早期集群监控和性能分析工具Heapster](plugins/Heapster/Heapster.md)(不使用)1~~
    - [prometheus](plugins/prometheus/prometheus.md)
- [外部服务暴露工具: Ingress](plugins/ingress/ingress.md)
- [包管理工具Helm](plugins/Helm/helm.md)
- [分布式存储系统Ceph](plugins/ceph/ceph.md)



## 其他资料
[k8s安装部署摘要](./k8s集群部署操作手册.pdf)
[dokcer-k8s-elk一站式](https://www.qikqiak.com/k8s-book/)


## 常用命令

```bash
# 查看集群node数量
kubectl get nodes
# 获取node详细信息
kubectl describe node <node_name>
# 查看所有pod运行状态
kubectl get pods --all-namespaces
kubectl get pod --all-namespaces -o wide
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
```
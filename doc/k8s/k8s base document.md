# k8s的一些基础概念


## 常用对象
- RC(Replication Controller)、RS(Replica Set): RC可以保证在任意时间内运行`Pod`的副本数量, 能够保证`Pod`总是可用的.
RS与RC的基本功能一致, 目前惟一的区别在于不同于RC只支持基于等式的`selector`, RS还支持基于集合的`selector`.
- Deployment: 主要职责与RC相同, 保证Pod的数量和健康. 除此之外还拥有以下功能
    - 事件和状态查看: 可以查看`Deployment`的升级详细进度和状态
    - 回滚: 当升级Pod出现问题, 可以使用回滚操作回滚到之前的任一版本
    - 版本记录: 每一次对`Deployment`的操作, 都能够保存下来, 这也是保证可以回滚到任一版本的基础
    - 暂停和启动: 对于每一次升级都能够随时暂停和启动
    ![](../../doc/picture/k8s/k8s%20base/deployment.png)
- HPA(Horizontal Pod Autoscaling): Pod水平自动伸缩. 当创建HPA后, HPA会从`Heapster`或者用户自定义的`RESTClient`端获取
每一个Pod的利用率并与HPA中定义的指标进行对比, 从而决定是否弹性伸缩. 
- Job、CronJob: Job仅执行一次的任务, 它保证批处理任务的一个或多个`Pod`成功结束. 而CronJob则就是在Job上加了时间调度.
- Service: Service是一种抽象的对象, 它定义了一组`Pod`的逻辑集合和一个用于访问它们的策略
    - kubernetes中的三种IP:
        - Node IP: Node节点的IP地址. 是k8s集群中节点的物理网卡IP地址(一般为内网), 集群内之间可直接通信, 集群外想要访问
        k8s集群内的节点或服务必须通过Node IP进行通信(这时一般是通过外网IP)
        - Pod IP: Pod的IP地址. 是每个Pod的IP地址
        - Cluster IP: Service的IP地址. 是一个虚拟的IP, 仅仅作用于`kubernetes Service`对象
            -  每个`Node`会运行一个`kube-proxy`进程, 监视k8s master对Service对象和Endpoints对象的添加和移除. 为Service
            实现Cluster IP的代理形式.
            ![](../../doc/picture/k8s/k8s%20base/services-iptables-overview.png)
            - Service类型:
                - ClusterIP: 通过集群的内部 IP 暴露服务,选择该值,服务只能够在集群内部可以访问,这也是默认的ServiceTyp
                - NodePart: 通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务.NodePort 服务会路由
                到 ClusterIP 服务,这个 ClusterIP 服务会自动创建.通过请求, 可以从集群的外部访问一个 NodePort 服务
                - LoadBalancer: 使用云提供商的负载均衡器,可以向外部暴露服务.外部的负载均衡器可以路由到 NodePort 
                服务和 ClusterIP 服务
                - ExternalName: 通过返回 CNAME 和它的值,可以将服务映射到 externalName 字段的内容(例如, 
                foo.bar.example.com). 没有任何类型代理被创建,这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持
- ConfigMap: ConfigMap提供了向容器中注入配置信息的能力, 不仅可以用来保存单个属性, 也可用来保存整个文件.
- Secret: Secret类似于Config的作用, 不同于Secret的明存储, 它是用来保存敏感信息(例如密码、ssh key等). 它有三种类型
    - Opaque: base64 编码格式的 Secret, 用来存储密码、密钥等; 但数据也可以通过base64-decode
    解码得到原始数据,所以加密性很弱
    - kubernetes.io/dockerconfigjson: 用来存储私有docker registry的认证信息
    - kubernetes.io/service-account-token: 用于被serviceaccount引用, serviceaccout 创建时Kubernetes会默认创建对应的
    secret.Pod如果使用了serviceaccount,对应的secret会自动挂载到Pod目录/run/secrets/kubernetes.io/serviceaccount中
- RBAC: 基于角色的访问控制. `RBAC`使用`rbac.authorization.k8s.io`API Group来实现授权决策
    - Rule: 规则(一组属于不同API Group资源上的一组操作的集合)
    - Role和ClusterRole: 角色和集群角色
        - Role定义的规则只适合单个命名空间
        - ClusterRole是集群范围内的
    - Subject: 主题, 对应在集群中尝试操作的对象, 集群中定义了三种类型的主题资源
        - User Account: 用户, 由外部独立服务进行管理, 由管理员进行私钥的分配.
        - Group: 组, 用来关联多个账户的, 集群中有些默认创建的组(比如cluster-admin)
        - Service Account: 服务账号, 通过`kubernetes`API来管理一些用户账号.
- RoleBinding和ClusterRoleBinding: 角色绑定和集群角色绑定, 简单来说就是把声明的subject和我们的Role进行绑定的过程(
给某个用户绑定上操作的权限)
- DaemonSet和StatefulSet: 
    - DaemonSet用于在每个`kubernetes`节点中将守护进程的副本作为后台进程运行. 在每个节点上部署一个Pod副本, 当节点加入
    集群时, Pod会被调度到该节点上运行; 节点移除后, 相应的Pod也会被移除.
    - StatefulSet类似于ReplicaSet, 但是它可以处理Pod的启动顺序, 为保留每个Pod的状态设置唯一标识, 同时具有以下功能:
        - 稳定的、唯一的网络标志符
        - 稳定的、持久化的存储
        - 有序的、优雅的部署和缩放
        - 有序的、优雅的删除和终止
        - 有序的、自动滚动更新
        
        
## 示例
[k8s部署Wordpress示例](https://www.qikqiak.com/k8s-book/docs/31.部署%20Wordpress%20示例.html): 
[wordpress-all.yaml](../../doc/k8s/example/wordpress-all.yaml)
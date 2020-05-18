# k8s cluster build

[原文链接](https://my.oschina.net/baobao/blog/3031712)

## 环境配置
一共五台虚拟机, 192.168.80.135-136为master节点, 192.168.80.137-139为worker节点

**Step1**: 创建虚拟机
- ubuntu16.04+
- 2GB+ RAM per machine. Any less leaves little room for your apps.(can low than that)
- 2 CPU or more on the control-plane node
- Full network connectivity between all machines in the cluster (public or private network is fine)
- Unique hostname, MAC address, and product_uuid for every node.
    ```bash
      # 检查MAC address唯一性
      ifconfig -a
      # 检查product_uuid唯一性
      sudo cat /sys/class/dmi/id/product_uuid
    ```
- Certain ports are open on your machines.
    ![](../../doc/picture/k8s/check%20ports.png)
- Swap disabled. You MUST disable swap in order for the `kubelet` to work properly
    ```bash
    # 临时关闭
    swapoff -a
    # 或永久关闭
    vim /etc/fstab
    #UUID=5d5184b2-33b0-493e-937d-88b91f76e383 none            swap    sw              0       0
    ```
    ![](../../doc/picture/k8s/swap%20off.png)

**Step2**: 修改节点名称
- 对于master节点
    - 修改`/etc/hostname`将`ubuntu`修改成`ubuntu-master`
    ```bash
      # 也可用命令
      hostnamectl --static set-hostname ubuntu-master-one
    ```
    - 修改`/etc/hosts`
    ```bash
    # 将`127.0.1.1  ubuntu`修改成
    127.0.1.1 hostname(ubuntu-master-one)
    192.168.80.135  ubuntu-master-one
    192.168.80.136  ubuntu-master-two
    192.168.80.137  ubuntu-worker-node-1
    192.168.80.138  ubuntu-worker-node-2
    192.168.80.139  ubuntu-worker-node-3
    
    # master节点域名解析涉及负载均衡(本示例并没有实现LB, 这边只用了两台, 生产环境中至少三台) 
    192.168.80.135  k8s.swh.com
    192.168.80.136  k8s.swh.com
    ```
    ![](../../doc/picture/k8s/hosts%20change.png)
  
- 同理, 对worker node节点做同样修改, 名称可随意(如: ubuntu-node-1)

**Step3**: 网络相关配置
- 添加路由转发
    ```bash
    # vim /etc/sysctl.d/k8s.conf
        # 配置转发参数
        net.bridge.bridge-nf-call-ip6tables=1
        net.bridge.bridge-nf-call-iptables=1
        net.ipv4.ip_forward=1
        
    # 执行以下命令生效
    modprobe br_netfilter
    sysctl -p /etc/sysctl.d/k8s.conf
    ```


## 安装k8s集群

**Step1**: Master节点搭建
- 安装docker, docker安装[脚本](../../doc/docker/docker.sh)
- 安装kubeadm、kubelet、kubectl
    ```bash
    apt-get update
    apt-get install -y apt-transport-https curl
    # google 官方源
        # 如果出现gpg: no valid OpenPGP data found错误. 是因为需要翻墙下载apt-key.gpg. 
        # 可选择访问https://packages.cloud.google.com/apt/doc/apt-key.gpg手动下载gpg文件. 地址: doc/kubernetes/files下
        # 再执行apt-key add apt-key.gpg
    # curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    # 阿里源
    echo "deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main" >> /etc/apt/sources.list
    # 安装kubeadm等
    apt-get update
    # google 所用
    # apt-get install -y kubelet kubeadm kubectl
    apt-get install -y kubelet kubeadm kubectl --allow-unauthenticated
    # hold kubelet kubeadm kubectl的版本号, 不更新, 可使用unhold取消
    apt-mark hold kubelet kubeadm kubectl
    # 重启kubelet
    systemctl daemon-reload
    systemctl restart kubelet
    systemctl status kubelet
    ```
- 单机上安装k8s, 先使用[kubeadm-config.yaml](../../doc/k8s/kubeadm-config.yaml)文件下载所需镜像
    ```bash
    # 使用命令
    kubeadm config images pull --config kubeadm-config.yaml
    kubeadm init --config kubeadm-config.yaml
    # kubeadm init --image-repository gcr.azk8s.cn/google_containers --kubernetes-version v1.15.0 --pod-network-cidr=192.168.0.0/16
    ```
    - 成功后, 保留kubeadm join,![](../../doc/picture/k8s/k8s%20init.png)
        ```bash
        # master 节点加入
        kubeadm join k8s.swh.com:6443 --token qqvegq.fpua47cda12fho28 \
            --discovery-token-ca-cert-hash sha256:80b090b1714afe853375a43d61fd301ae77d5892b06ad8e3ec77f03989bad155 \
            --experimental-control-plane
        # 节点加入
        kubeadm join k8s.swh.com:6443 --token qqvegq.fpua47cda12fho28 \
            --discovery-token-ca-cert-hash sha256:80b090b1714afe853375a43d61fd301ae77d5892b06ad8e3ec77f03989bad155 
        ```
    - 将KUBECONFIG置入环境变量
        ```bash
        # 临时
        export KUBECONFIG=/etc/kubernetes/admin.conf
        # 永久
        echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /root/.bashrc
        source /root/.bashrc
        ```
- 安装calico网络插件
    ```bash
    wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
    # 修改文件中的"value: "192.168.0.0/16"" 与kubeadm-config.yaml命令中的podSubnet一致
    kubectl apply -f calico.yaml
    ```

**Step2**: Master集群构建
- 新节点参照上述步骤安装好docker、kubeadm、kubelet、kubectl后
- 使用[脚本](../../doc/k8s/sync.master.ca.sh)将ca证书从源节点拷贝至其他节点
- 使用kubeadm join命令将节点加入master集群
    ```bash
    # 使用token作为control node加入k8s master 集群
      # kubeadm join master
    ```
- 添加环境变量
    ```bash
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /root/.bashrc
    source /root/.bashrc
    ```

**Step3**: Worker节点集群搭建
- 新节点参照Master节点初始化
- 使用kubeam join命令将节点加入至集群
    ```bash
    # 使用token作为worker node 加入k8s worker集群
        # kubeadm join worker node
    # 使用scp命令将master节点的admin.conf拷贝至worker 节点, 以便使用kubectl工具(可选)
    scp /etc/kubernetes/admin.conf  root@192.168.80.137:/etc/kubernetes/
    # 在worker节点上设置环境变量
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /root/.bashrc
    source /root/.bashrc
    ```

**Step4**: 安装必要插件
- [安装Dashboard插件](plugins/dashboard/dashboard.md)
- [集成Heapster监控及性能分析工具](plugins/Heapster/Heapster.md)


至此, 集群及必要插件已经搭建完成. ![](../../doc/picture/k8s/k8s%20complete.png)


## 其他的一些设置(可选)
创建k8s集群管理用户: 
- 使用[dashboard.yaml](admin.yaml)创建
    ```bash
    # 创建集群管理用户admin
    kubectl apply -f dashboard-admin.yaml
    # 查看admin的token信息
    kubectl get secret -n kube-system|grep admin-token
    # 获取指定token信息
    kubectl get secret admin-token-txcx4 -o jsonpath={.data.token} -n kube-system |base64 -d
    ```
    ![](../../doc/picture/k8s/service%20account.png)



## 遇到的一些问题
- [root用户无法远程连接解决方案](https://blog.csdn.net/qq_35445306/article/details/78771398)
    - 修改 `/etc/ssh/sshd_config` 文件把`PermitRootLogin Prohibit-password`添加#注释掉, 并添加`PermitRootLogin yes`.
    重启ssh`/etc/init.d/ssh restart`
    - 同时使用命令`passwd root`设置root命令
    
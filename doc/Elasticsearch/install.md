# Elastic Search Install

## 单机安装

### 压缩包安装
[下载地址](https://www.elastic.co/cn/downloads/elasticsearch)
```bash
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.2.0-linux-x86_64.tar.gz
tar -xvf elasticsearch-7.2.0-linux-x86_64.tar.gz
cd elasticsearch-7.2.0/bin
./elasticsearch
```

### apt-get 安装
```bash
# 安装依赖(可先进行执行安装elasticsearch再根据提示安装)
apt-get install libservlet2.5-java groovy2 libelasticsearch1.7-java
# 安装elasticsearch
sudo apt-get update && sudo apt-get install elasticsearch
```
~~未完待续...~~

## Kubernetes安装

### install with Docker(单节点)
```bash
# 下载镜像
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.2.0
```

- 开发者环境
```bash
# 运行
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.2.0
```
- 生产环境
    - `vm.max_map_count` kernel setting在生产环境中至少需要设置为`262144`
    ```bash
    # 修改/etc/sysctl.conf
    vm.max_map_count=262144
    
    # 启用该设定
    sysctl -w vm.max_map_count=262144
    ```
    - 使用docker compose工具根据`docker compose.yml`文件启动单节点上的物理集群(两个Elasticsearch node)。
    [docker compose 安装](#Docker Compose安装)
        - 创建的两个节点, `es01`监听在`localhost:9200`, 同时`es02`通过Docker network与`es01`沟通
        - [docker-compose.yml](dcoker-compose.yml)
        ```bash
        # 启动cluster命令(启动docker-compose up 之前, 现在执行该命令的目录下编写docker-compose.yml文件)
        docker-compose up
        # 检查cluster状态
        curl http://127.0.0.1:9200/_cat/health
        # 停止cluster命令
        docker-compose down
        # 停止cluster并销毁data volumes
        docker-compose down -v
        ```


Configuring Elasticsearch with Docker: 配置文件所在路径`/usr/share/elasticsearch/config/`, 这些配置文件详情
请查阅[配置文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)和
[Setting JVM options](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html)
    

## Docker Compose安装
[官网文档](https://docs.docker.com/compose/install/)
```bash
# 下载最新的release版本
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# 添加执行权限
sudo chmod +x /usr/local/bin/docker-compose
# 检查安装是否完成
docker-compose --version


# 如果docker-compose命令在安装后失败, 请检查路径动态链接至/usr/bin目录下
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

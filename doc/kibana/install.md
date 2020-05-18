# Kibana Install

## 单机安装

### 压缩包安装
[下载地址](https://www.elastic.co/cn/downloads/kibana)

### apt 安装
```bash
# 获取公钥
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
# 安装apt-transport-https工具包
sudo apt-get install apt-transport-https
# 添加APT repository源
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
# 安装kibana
sudo apt-get update && sudo apt-get install kibana
# 下载并安装the Debain package manually
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.2.0-amd64.deb
shasum -a 512 kibana-7.2.0-amd64.deb 
sudo dpkg -i kibana-7.2.0-amd64.deb
```
~~未完待续...~~

## Kubernetes安装

### Docker 安装

```bash
# 下载镜像
docker pull docker.elastic.co/kibana/kibana:7.2.0
# 运行镜像(开发环境)?未运行成功, kibana host 未成功运行
# docker run --link YOUR_ELASTICSEARCH_CONTAINER_NAME_OR_ID:elasticsearch -p 5601:5601 {docker-repo}:{version}
docker run --link es01:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:7.2.0
```

Configuring Kibana on Docker: Docker images提供了多种方法以配置Kibana
- 通用方法是提供`kibana.yml`文件作为Kibana配置文件
```bash
# docker-compose.yml
version: '2'
services:
  kibana:
    image: docker.elastic.co/kibana/kibana:7.2.0
    volumes:
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml
```
- 以环境变量的方式进行配置(但不灵活)
```bash
# docker-compose.yml
# 部分示例
version: '2'
services:
  kibana:
    image: docker.elastic.co/kibana/kibana:7.2.0
    environment:
      SERVER_NAME: kibana.example.org
      ELASTICSEARCH_HOSTS: http://elasticsearch.example.org
```
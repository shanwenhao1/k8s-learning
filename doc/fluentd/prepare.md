# Flume相关知识点

## 系统要求
- Java运行环境 - Java 1.8 or later
- Memory - sources、channels、sinks所配置的足够内存
- Disk Space - channels、sinks所配置的足够空间
- Directory Permissions - agent使用的文件夹的读写权限

## Fluentd介绍
- [Fluentd官方文档](https://docs.fluentd.org/)
- [Fluentbit](https://docs.fluentbit.io/manual/): 相对于fluentd更加轻量级, 资源占用量更少

### Fluent配置

名词解释:
- source: 指定数据源
- match: 指定输出地址
- filter: 指定了一个事件处理过程
- system: 用来设置系统的配置
- label: 为output和filter分组
- @include: 使用它可以在配置文件中包含其他的配置文件

具体情景配置: 以两个输入源(tcp, http), 一个输出(输出至file)为例
```bash
# Receive events from 20000/tcp
# This is used by log forwarding and the fluent-cat command
<source>
  @type forward
  port 20000
</source>

# http://this.host:8081/myapp.access?json={"event":"data"}
<source>
  @type http
  port 8081
</source>

# Match events tagged with "myapp.access" and
# store them to /var/log/td-agent/access.%Y-%m-%d
# Of course, you can control how you partition your data
# with the time_slice_format option.
<match myapp.access>
  @type file
  path /var/log/td-agent/access
</match>
```

## 架构

### 数据流模型
Data flow model
![](../picture/prepare/flume%20agent.png)

Agent主要由: source, channel, sink三个组件组成.
- source:
- channel:
- sink: 

### 可靠性
事件在每个代理的通道中进行, 然后将事件传递到流中的下一个代理或终端存储库(如HDFS). 只有事件存储在下一个代理或
终端存储库中时,通道中的事件才会被删除.

flume使用事务来保证事件的可靠传递

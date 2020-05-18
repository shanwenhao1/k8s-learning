# Flume相关知识点

## 系统要求
- Java运行环境 - Java 1.8 or later
- Memory - sources、channels、sinks所配置的足够内存
- Disk Space - channels、sinks所配置的足够空间
- Directory Permissions - agent使用的文件夹的读写权限


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

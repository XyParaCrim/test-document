# elasticsearch部署

## 使用的系统软件

| 名称 | 描述 |  版本 |
| ---| ---| --- |
|centos| 操作系统 | 7.2.1511
|elasticsearch|索引|7.3.0

## 运行环境

- JDK11(elasticsearch自带)

## 磁盘目录

| 说明 | 路径 |
| ---| ---|
| 解压安装包位置 | `/home/ifcp`
| 数据目录| `/home/ifcp/elasticsearch-7.3.0/data`
| 运行配置目录| `/home/ifcp/elasticsearch-7.3.0/config`
| 日志|`/home/ifcp/elasticsearch-7.3.0/logs`

## 安装并部署

### 解压安装
1. 解压至/home/ifcp`tar -xzvf elasticsearch-7.3.0-linux-x86_64.tar.gz -C /home/ifcp/`
2. 安装中文分词器
```bash
tar -xvf elasticsearch-analysis-ik-7.3.0.tar - C /home/ifcp/elasticsearch-7.3.0/plugins
mv /home/ifcp/elasticsearch-7.3.0/plugins/elasticsearch-analysis-ik-7.3.0 /home/ifcp/elasticsearch-7.3.0/plugins/ik
```

### 配置

#### elasticsearch.yml重要配置

- `cluster.name` - 集群名
- `node.name` - 节点名
- `node.master` - 是否有资格成为master
- `node.data` - 是否保存数据
- `node.voting_only` - 仅作为投票角色
- `network.host` - 绑定地址，填写本机地址
- `cluster.initial_master_nodes` - 节点名列表，启动时，将从列表中选出master
- `discovery.seed_hosts` - 地址列表，发现节点地址列表

#### master elasticsearch

```yaml
# /home/ifcp/elasticsearch-7.3.0/config/elasticsearch.yml
# 可以只留下面的内容
cluster.name: elastic
node.name: master
node.master: true
node.data: true
http.port: 9200
network.host: xx.xx.xx.xx # 填写本机地址
cluster.initial_master_nodes: ["master", "backup", "vote-only"]
discovery.seed_hosts: ["master:9300", "backup:9300", "vote-only:9300"] # 发现节点地址列表，这里用域名代替
```

#### backup elasticsearch

```yaml
# /home/ifcp/elasticsearch-7.3.0/config/elasticsearch.yml
# 可以只留下面的内容
cluster.name: elastic
node.name: backup
node.master: true
node.data: true
http.port: 9200
network.host: xx.xx.xx.xx # 填写本机地址
cluster.initial_master_nodes: ["master", "backup", "vote-only"]
discovery.seed_hosts: ["master:9300", "backup:9300", "vote-only:9300"] # 发现节点地址列表，这里用域名代替
```
#### vote-only elasticsearch

[因为在elasticsearch7.3版本中，如果只有2个节点，那么必须两者可用](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/modules-discovery-voting.html#modules-discovery-voting)。
所以添加一个voting_only角色的节点，功能仅用于master宕机时投票，保证集群可用。

```yaml
# /home/ifcp/elasticsearch-7.3.0/config/elasticsearch.yml
# 可以只留下面的内容
cluster.name: elastic
node.name: vote-only
node.master: true
node.voting_only: true
node.data: false # 不保存数据
http.port: 9200
network.host: xx.xx.xx.xx # 填写本机地址
cluster.initial_master_nodes: ["master", "backup", "vote-only"]
discovery.seed_hosts: ["master:9300", "backup:9300", "vote-only:9300"] # 发现节点地址列表，这里用域名代替
```
#### jvm options

```text
# /home/ifcp/elasticsearch-7.3.0/config/jvm.options
# 修改-XX:UseConcMarkSweepGC为-XX:+UseG1GC，去除警告
-XX:+UseG1GC
```

## 启动

```bash
/home/ifcp/elasticsearch-7.3.0/bin/elasticsearch
```
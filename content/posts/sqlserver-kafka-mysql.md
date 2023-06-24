---
title: "SQLServer数据实时同步到MySQL"
date: 2022-10-03T10:21:24+08:00
draft: true
tags:
  [
    "database",
    "distribte storage",
    "storage"
    "kafka"
  ]
categories: ["database","数据异构"]
---

## SQLServer 开启CDC

### 1. 确保SQL Server Agent服务开启

### 2. 添加CDC专用的文件组和文件

### 3. 开启库级别的CDC

### 4. 开启表级别的CDC

- 开启表级别cdc

```
IF EXISTS(SELECT 1 FROM sys.tables WHERE name='table_name' AND is_tracked_by_cdc = 0)
BEGIN
    EXEC sys.sp_cdc_enable_table
        @source_schema = 'dbo', -- source_schema
        @source_name = 'table_name', -- table_name
        @capture_instance = NULL, -- capture_instance
        @supports_net_changes = 1, -- supports_net_changes
        @role_name = NULL, -- role_name
        @index_name = NULL, -- index_name
        @captured_column_list = NULL, -- captured_column_list
        @filegroup_name = 'CDC' -- filegroup_name
END
```

- 关闭表级别cdc

```
USE databasename123; 
GO 
EXECUTE sys.sp_cdc_disable_table 
@source_schema = N'dbo', 
@source_name = N'UserTable', 
@capture_instance = N'ALL';
```

## 容器化安装Apache Kafka

1. 准备数据目录

``` bash
# 创建目录
$ mkdir  kafka_data  kafka_plugins  zookeeper_data

# 容器默认用户为1001,修改属主
$ chown 1001:1001 -R  kafka_data  kafka_plugins  zookeeper_data
```

2. 使用docker-compose安装zookeeper和kafka

```
version: "2"

networks:
  dnet:
    driver: bridge

services:
  zookeeper:
    image: docker.io/bitnami/zookeeper:3.8
    ports:
      - "2181:2181"
    volumes:
      - "./zookeeper_data/:/bitnami:rw"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    networks:
      - dnet
  kafka:
    image: docker.io/bitnami/kafka:2.8.1
    ports:
      - "9092:9092"
    volumes:
      - "./kafka_data/:/bitnami/kafka:rw"
      - "./kafka_plugins/:/bitnami/plugins/:rw"
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://192.168.30.50:9092 
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper
    networks:
      - dnet
  kafka-ui:
    image: wdkang/kafka-console-ui:latest
    ports:
    - "7766:7766"
    depends_on:
      - kafka
    networks:
      - dnetw

```

3. 启动zookeeper, kafka, kafka-ui 容器

```
docker-compose up -d 
```

4. 此时可以通过<http://192.168.30.50:7766/#/>访问kafka console ui

## 安装配置Kafka Connect

1. 添加docker-compose文件如下内容

```
 kafka-connect:
   image: docker.io/bitnami/kafka:2.8.1
   ports:
     - "8083:8083"
    depends_on:
      - kafka
    networks:
      - dnet
    volumes:
      - "./kafka_connect_plugins/:/bitnami/connect/plugins/:rw"
      - "./kafka_connect_configs/:/opt/bitnami/kafka/config:rw"
    environment:w
      - CLASSPATH=/bitnami/connect/plugins/debezium-connector-sqlserver/*:/bitnami/connect/plugins/debezium-connector-jdbc/*:/bitnami/connect/plugins/confluentinc-kafka-connect-jdbc-10.7.3/lib/*:/bitnami/connect/plugins/confluentinc-kafka-connect-protobuf-converter-7.4.0/lib/*
   command: ["/opt/bitnami/kafka/bin/connect-distributed.sh", "/opt/bitnami/kafka/config/connect-distributed.properties"]

```

2. 准备初始配置文件

```
for f in `docker exec -u root  kafka_kafka_1 ls /opt/bitnami/kafka/config`;do docker cp  kafka_kafka_1:/opt/bitnami/kafka/config/${f} . ;done
```

3. 修改connect-distributed.properties文件

```
bootstrap.servers=kafka:9092

group.id=qlk-connect-cluster

key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true

internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter.schemas.enable=false
internal.value.converter.schemas.enable=false

offset.storage.topic=connect-offsets
offset.storage.replication.factor=1
offset.storage.partitions=2

config.storage.topic=connect-configs
config.storage.replication.factor=1
errors.log.enable=true

status.storage.topic=connect-status
status.storage.replication.factor=1
status.storage.partitions=2

# Flush much faster than normal, which is useful for testing/debugging
offset.flush.interval.ms=10000

#rest.host.name=
rest.port=8083

# The Hostname & Port that will be given out to other workers to connect to i.e. URLs that are routable from other servers.
#rest.advertised.host.name=
#rest.advertised.port=

# plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,


plugin.path=/bitnami/connect/plugins/
```

4. 准备connector plugin的jar包

- 从 <https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc> 下载confluentinc-kafka-connect-jdbc, 这个包会包括source connector和sink connector， 将下载的压缩包解压到kafka_connect_plugins目录

```
unzip confluentinc-kafka-connect-jdbc-10.7.3.zip
```

- 从<https://www.confluent.io/hub/confluentinc/kafka-connect-protobuf-converter> 下载kafka-connect-protobuf-converter， 将下载的压缩包解压到kafka_connect_plugins目录

```
unzip confluentinc-kafka-connect-protobuf-converter-7.4.0.zip
```

- 下载Debezium的sql server connector 和 jdbc sink connector（可选）

5. 启动kafka-connect container

```
docker-compose up -d 
```

## 安装 confluent schema registry

1. 下载社区开源版本的TConfluent Platform
   <https://www.confluent.io/installation/>

2. 配置 Confluent Schema Registry 服务

- 下载好后上传到服务器，解压即可用,这儿是confluent-7.4.0目录
- 进入confluent-7.4.0/etc/schema-registry/目录下，修改- schema-registry.properties文件，内容及注释如下：

```
# Confluent Schema Registry 服务的访问IP和端口
listeners=http://192.168.101.50:8081

# Kafka集群的地址
kafkastore.bootstrap.servers=PLAINTEXT://192.168.101.50:9092

# 存储 schema 的 topic
kafkastore.topic=_schemas

# 其余保持默认即可

```

3. 启动 Confluent Schema Registry 服务

```

# 进入到confluent-7.4.0/bin目录执行
bin/schema-registry-start etc/schema-registry/schema-registry.properties

```

此次也可以使用docker部署方式
<https://docs.confluent.io/platform/current/platform-quickstart.html>

## connector实例执行数据同步

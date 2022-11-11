---
title: docker测试环境搭建
tags:
  - Docker
date: 2022-08-24
categories:
  - 后端
---

## 环境
- 工具：docker，docker-compose

## 需求描述
在本地搭建业务中可能使用的服务，方便本地进行测试。

## 配置文件

```yaml
version: "3"
services:
  zookeeper:
    container_name: zookeeper
    image: 'bitnami/zookeeper:latest'
    ports:
      - '2181:2181'
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
  kafka:
    container_name: kafka
    image: 'bitnami/kafka:latest'
    ports:
      - '9092:9092'
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper
    volumes: 
      - 'kafka:/bitnami/kafka'
  redis:
    container_name: redis
    image: 'redis:latest'
    ports: 
      - '6379:6379'
    volumes: 
      - 'redis:/usr/local/etc/redis'
  mysql:
    container_name: mysql
    image: 'mysql:latest'
    ports: 
      - '3306:3306'
    environment:
        MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
        MYSQL_DATABASE: app
    volumes: 
      - 'mysql:/var/lib/mysql'
  mongo:
    container_name: mongo
    image: 'mongo:latest'
    ports: 
      - '27017:27017'
    volumes: 
      - 'mongo:/data/db'
  etcd:
    container_name: etcd
    image: 'quay.io/coreos/etcd:latest'
    ports: 
      - '2379:2379'
      - '2380:2380'
    volumes: 
      - 'etcd:/etcd-data'
    command: 
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name=node1 
      - --initial-advertise-peer-urls=http://192.168.82.116:2380 
      - --listen-peer-urls=http://0.0.0.0:2380 
      - --advertise-client-urls=http://192.168.82.116:2379 
      - --listen-client-urls=http://0.0.0.0:2379 
      - --initial-cluster 
      - node1=http://192.168.82.116:2380
  es:
    container_name: es
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    environment:
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es:/usr/share/elasticsearch/data
    ports:
      - '9200:9200'

volumes: 
  redis: {}
  mysql: {}
  kafka: {}
  es: {}
  mongo: {}
  etcd: {}

```
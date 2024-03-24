---
title:        Dive into Flink (2)
date:         2024-03-23
categories:   [Data, Engineering]
comments:     true
---

<style>
H2 { color: #298294 }
H3 { color: #1e7ed2 }
H4 { color: #C7A579 }
</style>

## WIP
이전 글에 이어 이번에는 Flink를 띄워보고 애플리케이션을 구현해보려고 한다.


## Practice

1. FastAPI로 구현한 Application에서 HTTP 요청이 발생하면 데이터를 비동기적으로 Kafka로 전송
2. Flink Application은 토픽에 저장된 데이터를 MinIO에 스트리밍 방식으로 적재, Iceberg 사용
3. Trino는 MinIO에 저장된 데이터를 쿼리

![image_01](/assets/img/posts/2024-03-24/image_01.png){: width="600" height="400" }


### Docker Setting

> 다음 Github 링크에 설정들을 상세하게 정리해 두었습니다.
> - [https://github.com/ivoryRabbit/play-data-with-docker/tree/master/flink](https://github.com/ivoryRabbit/play-data-with-docker/tree/master/flink){: target="_blank"}
{: .prompt-tip }



#### Kafka

실습이므로 카프카 브로커 개수는 1개, zookeeper를 띄우지 않기 위해 KRaft 지원이 가능한 3.4.1 버전 사용

```yaml
kafka:
  container_name: kafka
  image: bitnami/kafka:3.4.1
  ports:
    - "9094:9094"
  environment:
    # KRaft settings
    KAFKA_CFG_NODE_ID: 0
    KAFKA_CFG_PROCESS_ROLES: controller,broker
    KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
    # Listeners
    KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
    KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
    KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
    KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
  volumes:
    - ./docker/volume/kafka:/bitnami/kafka
```

#### Kafka UI

CLI 작성이 귀찮으므로 kafka UI 사용

`DYNAMIC_CONFIG_ENABLED`를 사용하면 클러스터 등록/제거가 편리

```yaml
kafka-ui:
  image: provectuslabs/kafka-ui
  container_name: kafka-ui
  ports:
    - "8080:8080"
  restart: always
  environment:
    - DYNAMIC_CONFIG_ENABLED=true
  volumes:
    - ./docker/kafka-ui/config.yml:/etc/kafkaui/dynamic_config.yaml
  healthcheck:
    test: wget --no-verbose --tries=1 --spider localhost:8080 || exit 1
    interval: 5s
    timeout: 10s
    retries: 3
    start_period: 30s
```

#### Flink

Flink는 jobmanager와 taskmanager를 각각 배포

```yaml
flink-jobmanager:
  container_name: flink-jobmanager
  hostname: flink-jobmanager
  image: flink:1.18.1
  ports:
    - "8081:8081"
  command: jobmanager
  environment:
    - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
  volumes:
    - ./docker/volume/flink/jobmanager:/data/flink

flink-taskmanager:
  container_name: flink-taskmanager
  hostname: flink-taskmanager
  image: flink:1.18.1
  command: taskmanager
  environment:
    - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
  volumes:
    - ./docker/volume/flink/taskmanager:/data/flink
```

## Applications

### Producer

Python
FastAPI
kafka-python

uvicorn
docker 배포

### Consumer

Scala
SBT
Flink API

개발 환경이 mac m1인데 로컬에서 pyflink가 정상적으로 동작하지 않아 scala를 사용하려 함


## Analysis

### Iceberg

### Trino

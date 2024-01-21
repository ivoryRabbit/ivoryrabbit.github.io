---
title:        Trino Gateway를 알아보자
date:         2024-01-20
categories:   [Data, Engineering]
comments:     true
---

<!-- <style>
H2 { color: #d2691e }
H3 { color: #cd853f }
H4 { color: #deb887 }
</style> -->

## Trino란?

Trino는 대용량 데이터 셋을 여러 서버에 걸쳐 병렬로 처리하기 위한 분산 쿼리 엔진으로, HDFS 뿐만 아니라 MySQL, Kafka, Cassandra 등 다양한 종류의 데이터 소스를 지원하여 많은 사람들에게 사랑받고 있는 오픈 소스이다.

Trino Cluster는 하나의 Coordinator와 다수의 Workers로 구성되어 있는데, 사용자가 쿼리를 날리면 Coordinator가 전달 받은 SQL를 분석하여 실행 계획을 세운다. 이에 따라 Worker들이 데이터 소스에 접근하여 작업을 수행하고, 그 결과를 Coordinator를 통해 사용자에게 전달한다.

작업이 시작되면 하나의 쿼리는 여러 "stage"로 나뉘어 차례대로 실행되며, Coordinator는 Worker들과의 API 통신을 통해 작업 경과를 보고 받는다. 그리고 Spark와 마찬가지로 "Shuffle"을 위해 Worker 끼리 통신을 하기도 한다.

## Trino Gateway란?

Trino Gateway는 다수의 Trino Cluster를 운영할 때 유용한, 일종의 프록시 서버이다. Trino와 마찬가지로 Presto Gateway로부터 folk되어 개발되었으며, 자세한 내용은 Github에서 확인 가능하다.

- https://github.com/trinodb/trino-gateway

이름에서 유추할 수 있듯이 현업에서 다음과 같은 목적으로 사용될 수 있다.

### 1. Routing Gateway

분석 환경을 굳이 구분하자면, 적시적인 데이터 분석을 위해 상시로 쿼리가 실행되는 "Adhoc" 환경과 데이터 마트 또는 머신러닝을 위해 배포된 파이프라인이 실행되는 "Production" 환경 두 가지가 있을 수 있다.

두 개발 환경 모두가 같은 Trino Cluster를 사용하게 된다면, 특정 시간에 스케쥴링된 Production 환경의 작업들이 Adhoc하게 실행된 쿼리에 의해 방해 받을 수 있다.

이를 방지하기 위해서는 두 개 이상의 클러스터를 운영하여 각 환경이 목적에 맞는 Trino 서버에 접근할 수 있도록 분리하는 것이 좋다. 이때 Trino Gateway를 사용하면 사용자들에게는 하나의 URL만 제공하되 환경 별로, 혹은 사용자 별로 연결을 통제할 수 있기 때문에 관리에 용이하다.

### 2. Blue / Green Deployment

Trino Cluster를 사용하고 있다면 온디맨드 서비스를 제공하기 위해 서버가 상시적으로 띄워져 있을 가능성이 높다. 

이 때 클러스터를 업그레이드 하거나 새로운 노드로 교체하고자 한다면 Trino Gateway를 이용해 Blue / Green 배포가 가능하다. 새로운 클러스터 서버를 띄운 후, API를 통해 Trino Gateway에서 교체하고자 하는 서버와 같은 라우팅 그룹에 등록해주면 된다.

클러스터가 Trino Gateway에 한번 등록된 이후에도 API를 통해 해당 클러스터의 라우팅 그룹을 변경하거나 비활성화를 위해 Graceful Shutdown 시키는 것도 가능하다.

## 직접 띄워보기

Docker를 이용하여 로컬 개발 환경에서 Trino Gateway를 사용해볼 수 있다. Trino Gateway를 띄우려면 다수의 Trino Cluster가 필요하고, Trino Cluster를 띄우기 위해서는 데이터가 저장될 Object Storage와 Hive Metastore가 필요하다.

### 1. Trino

전체 아키택처를 간략하게 그리면 다음과 같다.

![image_01](/assets/img/posts/2024-01-21/image_01.png){: width="800" height="400" }

S3 대신 MinIO 사용

```yaml
...
  minio:
    container_name: minio
    hostname: minio
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
      MINIO_DOMAIN: minio
    command: server /data --console-address ":9001"
    restart: always
```

Postgres

```yaml
postgres:
    container_name: postgres
    hostname: postgres
    image: postgres:14-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./docker/volume/postgres:/var/lib/postgresql/data
      - ./docker/postgres/init-database.sh:/docker-entrypoint-initdb.d/init-database.sh
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "postgres" ]
      interval: 10s
      retries: 3
      start_period: 5s
```


Hive Metastore

```yaml
...

hive-metastore:
    container_name: hive-metastore
    hostname: hive-metastore
    image: starburstdata/hive:3.1.3-e.6
    ports:
      - "9083:9083"
    env_file:
      - ./docker/hive-metastore/.env
    depends_on:
      postgres:
        condition: service_healthy
```

1개의 Coordinator와 2개의 Worker로 Trino Cluster를 구성

```yaml
...

trino-1:
    container_name: trino-1
    hostname: trino
    image: trinodb/trino:435
    ports:
      - "8081:8080"
    volumes:
      - ./docker/trino/etc-coordinator:/etc/trino
      - ./docker/trino/catalog:/etc/trino/catalog
    depends_on:
      - hive-metastore

  trino-1-worker-1:
    container_name: trino-1-worker-1
    hostname: trino-worker-1
    image: trinodb/trino:435
    ports:
      - "8082:8080"
    volumes:
      - ./docker/trino/etc-worker:/etc/trino
      - ./docker/trino/catalog:/etc/trino/catalog
    depends_on:
      - trino-1
  
  trino-1-worker-2:
    container_name: trino-1-worker-2
    hostname: trino-worker-2
    image: trinodb/trino:435
    ports:
      - "8083:8080"
    volumes:
      - ./docker/trino/etc-worker:/etc/trino
      - ./docker/trino/catalog:/etc/trino/catalog
    depends_on:
      - trino-1
```

### 2. Trino Gateway

Trino Gateway 서버

```Dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /etc/trino-gateway

RUN apt-get -y update && apt-get -y install curl

ENV VERSION=4

RUN curl https://repo1.maven.org/maven2/io/trino/gateway/gateway-ha/${VERSION}/gateway-ha-${VERSION}-jar-with-dependencies.jar -o gateway-ha.jar
```

백엔드에 클러스터 등록

```json
{
    "name": "trino-1",
    "proxyTo": "http://trino-1:8080",
    "active": true,
    "externalUrl": "http://localhost:8081",
    "routingGroup": "adhoc"
}
```

```bash
curl -H "Content-Type: application/json" -X POST localhost:9080/gateway/backend/modify/update -d @scripts/register-trino-1.json
```

![image_02](/assets/img/posts/2024-01-21/image_02.png){: width="800" height="400" }
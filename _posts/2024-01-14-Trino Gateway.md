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

Trino Gateway는 다수의 Trino Cluster를 운영할 때 유용한, 일종의 프록시 서버이다. Trino Gateway 또한 Trino와 마찬가지로 Presto Gateway로부터 folk되어 리팩토링되었으며, 오픈소스이기 때문에 Github에서 찾아볼 수 있다.

- https://github.com/trinodb/trino-gateway

이름에서 유추할 수 있듯이 다음과 같은 목적으로 사용될 수 있다.

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

전체적인 아키택처는 다음과 같다.

![image_01](/assets/img/posts/2024-01-21/image_01.png){: width="800" height="400" }

### 1. Trino

먼저 데이터가 저장될 Object Storage를 구성해야 한다. 일반적으로 사용되는 AWS S3 대신 S3 SDK와 잘 호환되는 MinIO를 사용해보았다.

```yaml
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

다음은 관계형 데이터베이스인 Postgres를 구성한다. 이 데이터베이스는 Hive Metadata 및 Trino Gateway의 백엔드 역할을 하게된다.

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
```

다음은 Trino가 MinIO에 저장된 데이터를 조회할 수 있도록 테이블 및 파티션 정보를 저장할 Hive Metastore를 띄워 본다. `.env` 파일에 S3 엔드포인트와 Postgres 서버 정보를 환경 변수에 등록하여 Hive Metastore가 접근할 수 있도록 설정해준다.

```yaml
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
```

마지막으로 Trino Cluster를 세팅한다. 1개의 Coordinator와 2개의 Worker로 Trino Cluster를 구성한다. 같은 도커 이미지를 사용하되 config.properties 파일을 마운트하여 Coordinator와 Worker를 결정할 수 있다.

```yaml
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

좀 더 상세한 내용은 Github에 업로드해 두었다.

- https://github.com/ivoryRabbit/play-data-with-docker/tree/master/trino

### 2. Trino Gateway

Trino Gateway 서버는 JVM 기반으로 작동한다. Maven에 등록된 JAR 파일을 다운로드 받은 후 서버를 실행 시킨 뒤, 3개의 port를 열어준다. gageway-config.yaml 파일에는 백엔드로 사용할 Postgres 서버 정보가 포함되어 있다.

```Dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /etc/trino-gateway

RUN apt-get -y update && apt-get -y install curl

ENV VERSION=4

RUN curl https://repo1.maven.org/maven2/io/trino/gateway/gateway-ha/${VERSION}/gateway-ha-${VERSION}-jar-with-dependencies.jar -o gateway-ha.jar
```

```yaml
  trino-gateway:
    container_name: trino-gateway
    hostname: trino-gateway
    build:
      dockerfile: ./docker/trino-gateway/Dockerfile
    image: trino-gateway
    ports:
      - "9080:9080"
      - "9081:9081"
      - "9082:9082"
    volumes:
      - ./docker/trino-gateway/gateway-config.yaml:/etc/trino-gateway/gateway-config.yaml
    depends_on:
      postgres:
    entrypoint: >
      java -Xmx1g --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED -jar gateway-ha.jar server gateway-config.yaml
```

### 3. Rest API

Trino Cluster와 Trino Gateway 서버가 띄워졌다면 Rest API로 클러스터를 서버에 등록할 수 있다.

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

9080 port로 접속하면 등록된 클러스터를 웹서버를 통해 확인할 수 있다.

![image_02](/assets/img/posts/2024-01-21/image_02.png){: width="800" height="400" }
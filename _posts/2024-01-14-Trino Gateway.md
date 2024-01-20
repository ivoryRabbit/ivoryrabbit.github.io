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

Trino Cluster는 하나의 Coordinator와 다수의 Workers로 구성되어 있는데, 사용자가 쿼리를 날리면 Coordinator가 전달 받은 SQL를 분석하여 실행 계획을 세우고 이에 따라 Worker들이 데이터 소스에 접근하여 작업을 수행한다.

작업이 시작되면 하나의 쿼리는 여러 "stage"로 나뉘어 차례대로 실행되며, Coordinator는 Worker들과의 API 통신을 통해 작업 경과를 보고 받는다. 그리고 Spark와 마찬가지로 "Shuffle"을 위해 Worker 끼리 통신을 하기도 한다.

## Trino Gateway란?

Trino Gateway는 다수의 Trino Cluster를 운영할 때 유용한, 일종의 프록시 서버이다. 이름에서 유추할 수 있듯이 현업에서 크게 3가지 목적으로 사용되고 있다.

### 1. Load Balancing

분석 환경을 굳이 나누자면 적시적인 데이터 분석을 위해 상시로 쿼리가 실행되는 Adhoc 환경과 데이터 마트 및 머신러닝 작업을 위해 배포된 파이프라인이 실행되는 Production 환경, 두 가지로 분류할 수 있을 것이다.

두 개발 환경 모두 공통의 Trino Cluster를 사용하게 된다면, 특정 시간에 스케쥴링된 Production 환경의 작업들이 Adhoc하게 실행된 쿼리에 의해 방해 받을 수 있다.

이를 방지하기 위해 두 개 이상의 클러스터를 운영하여 각 환경이 목적에 맞는 Trino 서버에 연결할 수 있도록 구축하는 것이 좋다. 이때 Trino Gateway를 사용하면 사용자들에게는 하나의 URL만 제공하되 환경 별로 연결을 통제할 수 있기 때문에 관리하기에 용이하다.

### 2. Routing Gateway

사용자가 필요로하는 클러스터로만 라우팅하도록


### 3. Blue - Green Deployment

다운 타임 없이 버전을 업그레이드 하거나 새로운 노드로 교체할 때


## 직접 띄워보기

### 1. Trino

### 2. Trino Gateway

### 3. Rest API



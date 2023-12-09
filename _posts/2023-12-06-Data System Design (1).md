---
title:        데이터 시스템 디자인 (1)
date:         2023-12-09
categories:   [Data, Engineering]
comments:     true
---

## 들어가기 앞서

데이터 엔지니어의 역할은 사내 임직원들이 데이터를 활용하여 의사 결정할 수 있도록 빠르고 안정적이며 값싼(?) 분석 환경을 제공하는 것입니다.

따라서 데이터 엔지니어는 어떤 회사, 어떤 팀에 속하든지 간에 데이터를 수집 및 가공하는 기술을 바탕으로 업무를 수행하게 됩니다.

예를 들어 데이터 처리 파이프라인을 담당하는 데이터 엔지니어는 데이터를 처리하는 application이 실패하더라도 재실행을 통해 데이터가 충분히 복구될 수 있도록 로직을 구성해야 합니다.

또한 데이터 인프라를 담당하는 데이터 엔지니어는 데이터 처리 담당 클러스터가 데이터를 잃어버리는 일이 없도록 신뢰 가능하게(Reliable) 운영해야 합니다.

저는 이에 대한 내용을 제 경험을 바탕으로 한번 쯤 정리하면 좋겠다는 생각에 이 글을 작성하기로 하였습니다.

본 글에 들어가기 앞서 간단한 가정과 용어 정리를 하겠습니다.

### 가정
1. 서비스 환경과 분석 환경은 분리되어야 한다.
2. 분석 환경은 하나로 통일되어야 한다.

가정 1은 서비스 환경에 분석 쿼리를 날려서는 안된다는 것을 의미합니다. 분석 환경을 서비스 환경으로부터 분리하고, 서비스 환경에서 분석 환경으로 데이터를 흘려보내는 것이 데이터 엔지니어링이자 이 모든 비극의 시작입니다.

가정 2는 데이터를 충분히 잘 분석하기 위해서는 조인이 반드시 필요하며, 이를 위해선 단 하나의 논리적 공간에서 모든 데이터를 조회가능해야 함을 의미합니다. 이 글에서는 이러한 분석 환경을 DW(Data Warehouse)라고 부르겠습니다.

## Data Ingestion

Data Ingestion은 기존의 ETL(Extract-Transform-Load) 방식에서 분리되어, 데이터를 추출하여 DW로 전달하는데에만 집중하는 개념입니다.

데이터 추출이 필요한 source가 어떻게 생겼냐에 따라 다양한 Data Ingestion 방식이 존재하지만 데이터 엔지니어는 source를 결정할 수 없기에, 데이터 구조에 따라 다른 방식으로 target table을 디자인해야 합니다.

Target table을 디자인하는데는 다음과 같은 요소를 고려해야 합니다.

1. Mutable vs Immutable
2. Fact table vs Dimension table
3. Batch vs Streaming



각 data ingestion 마다 장단점 정리

단점을 극복하기 위해 다음 스탭으로 넘어가는 방식


### 1. Snapshot
RDB에 적재된 테이블을 통째로 snapshot 찍어 업로드

transaction data로, 테이블에서 업데이트가 일어나는 경우

dimension table 이거나
데이터 규모가 작거나
정산과 같이 분석 단에서 예전 데이터를 봐야만 하는 경우


### 2. Incremental
적재된 테이블이 insert만 발생하는 경우
로그
적재할 테이블을 partition으로 운영하되, 증분으로 데이터를 집어 넣어준다

데이터가 생성된 날짜를 가리키는 flag 필요
- source가 rdb라면 row가 생성된 created_at
- source가 stream 이라면 스토리지에 적재된 시간 필요

### 3. Delta
기존 테이블을 계속 유지하면서 source table의 변경 분만 가져다가 target table에 merge하는 방식

target은 source와 계속 sync가 맞춰짐

### 4. Delta + Incremental
테이블을 Delta로 운영하여 target table의 최신 상태를 유지하되, 중요한 event들은 변경이 일어날때마다 별도의 테이블에 incremental하게 업데이트

### 5. Snapshot + Delta
Snapshot을 찍되, Snapshot을 찍은 시점 이후의 Delta를 모아 다시 Snapshot을 생성하는 방식
source와 target이 비교적 낮은 latency로 sync될 필요가 있는 경우

### 6. Streaming
Insert만 발생하는 경우에 사용 가능

쿼리를 날리는 동시에 지금까지 모은 데이터를 consuming하는 방식
쿼리가 들어오지 않으면 지정해둔 파일 사이즈가 될 때까지 파일을 모았다가 자동으로 consume

### 7. Streaming + Delta (CDC)
데이터가 생성될 때 마다 원본 테이블에 Delta 데이터를 upsert
다만 추후 읽기 성능을 위해서 주기적으로 파일을 compact하게 정리해주어야 함

Apache Pinot


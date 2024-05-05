---
title:        System Design Interview (2)
date:         2024-05-05
categories:   [Data, Engineering]
math:         true
comments:     true
---

<style>
H2 { color: #1e7ed2 }
H3 { color: #298294 }
H4 { color: #C7A579 }
</style>

이전 글에서는 시스템 디자인이 왜 중요한지, 또 어떤 사항을 고려해야 할지에 대해 다루었다. 이번 글에서는 실제로 운영되고 있는 서비스들의 시스템 디자인을 탐구해봄으로써 간접적으로나마 설계 문제를 풀어나가는 능력을 키워보고자 한다.

## Design: TinyURL

TinyURL은 길이가 긴 URL을 보다 짧은 URL로 라우팅 해주는 서비스이다. 비슷한 서비스로는 bitly, rebrandly 등이 있다. 이러한 서비스를 이용하면 기나긴 URL을 줄일 수 있을 뿐만 아니라 링크를 추적하여 트래픽 및 마케팅 성과를 분석할 수 있으며, 원본 URL의 노출을 막을 수도 있다.

### Requirements and Goals

가장 먼저 이 서비스의 요구 사항을 정리해보자.

#### Functional Requirements

1. 사용자의 URL로부터 짧고 유일한 "short link"를 생성
2. 생성된 short link는 원래의 URL 링크를 redirection
3. 사용자는 short link를 직접 선택할 수도 있음
4. 생성된 링크는 만료 기간이 존재하며, 사용자가 이 기간을 지정하는 것도 가능

#### Non-functional Requirements

1. 시스템은 고가용성(High Availablility)이 필요함
2. Redirection는 짧은 지연시간 내에 이루어져야 함
3. 생성된 링크는 예측 불가능해야함

#### Extended Requirements

1. Re-direction이 얼마나 발생했는지 분석 가능
2. REST API를 통해 다른 서비스가 이 서비스에 접근 가능

### Capacity Estimation

다음은 시스템의 규모와 제약 사항을 살펴보자. 우리가 만들고자 하는 시스템은 분명 redirection 요청 횟수가 URL 생성 횟수보다 많을 것이다. 따라서 read:write 비율을 100:1 정도로 가정해보자.

#### Traffic Estimation

한달에 약 500M(5억)건의 URL이 새로 생성된다고 하자. 그러면 redirection은 한달에 50B(500억)회 정도 발생할 것이다. 이를 환산하면 대략 write는 200/sec, read는 20K/sec 정도로 어림잡을 수 있다.

#### Storage Estimation

새로 생성된 URL의 생애주기를 5년으로 잡자. 그러면 한달에 500M 건의 새로운 URL이 생성되므로 허용 가능한 URL의 개수는 다음과 같다.

$$ 500\text{M} \times 5\text{ year} \times 12\text{ month} = 30\text{B}$$

URL 하나의 object 크기를 500 bytes라 가정하면 데이터 저장에 필요한 총 용량은 15TB가 된다.

$$ 30\text{B} \times 500\text{ bytes} = 15\text{TB} $$

#### Bandwidth Estimation

위에서 예상한 트래픽은 write가 200/sec, read가 20K/sec 였다. 따라서 서비스에 들어오고 나가는 데이터의 총량은 다음과 같이 계산된다.

- write:
$$ 200\text{/sec} \times 500 \text{ bytes} = 100 \text{KB/sec}$$

- read:
$$ 20K\text{/sec} \times 500 \text{ bytes} = 10 \text{MB/sec}$$

#### Memory Estimation

지연시간을 줄이기 위해 캐싱을 적용한다고 해보자. 80:20 법칙을 적용하여 20%의 URL이 80% 트래픽을 발생시킨다고 했을 때, 자주 요청되는 20%의 URL을 캐시하려면 170GB 정도의 메모리가 필요하다.

$$20\text{K/sec} \times 3600 \text{ sec} \times 24\text{ hour} = 1.7\text{B/day}$$

$$0.2 \times 1.7\text{B} \times 500 \text{ bytes} = 170{GB}$$

중복 요청을 고려하면 실제 메모리는 이보다 더 적게 차지할 것이다.

### System API

SOAP 또는 REST API를 이용하여 이 서비스의 기능을 노출시킬 수 있다. URL을 생성하거나 제거하기 위한 API를 정의해보자.

#### createURL

- Request Parameters
    - api_dev_key: 개발자를 위한 등록된 계정의 API key
    - original_url: 원본 URL
    - custom_alias: 사용자 지정 short link
    - user_name: 필요시 인코딩에 사용자 이름을 넣을 수 있음
    - expire_date: short link의 만료 날짜

- Response
    - shorten_url: 생성된 short link의 URL
    - url_key: 생성된 short link의 식별자

#### deleteURL

- Request Parameters
    - api_dev_key: 개발자를 위한 등록된 계정의 API key
    - url_key: 생성된 short link의 식별자

- Response
    - 삭제 성공 여부




metadata를 저장할 database 설계

Key Generation Service Application

스케일을 고려하여 6자리의 문자열 생성

1. 매번 요청이 들어올때마다 md5 또는 sha256 적용 후 base64 인코딩 ([a-z0-9A-Z . -]) 하는 방법
2. 미리 키를 만들어놓고 요청마다 한개 씩 사용하는 방법 (unused key table, used key table)



## Design: Pastebin

글이나 이미지를 저장하고 url을 이용하여 공유할 수 있는 서비스

metadata를 저장할 database 설계

reliable을 위해 distributed object storage 사용 (ex: S3)

metadata database가 글이나 이미지가 저장된 object storage의 url을 갖고 있다가 요청이 들어오면 라우팅

트래픽이 많은 경우 미리 키를 만들어 놓는 것이 좋다 (unused key table, used key table)

테이블 샤딩도 가능. 2개의 서버에서 2씩 증가하면서 각각 offset이 1, 2가 되도록 데이터 저장

## Design: Instagram

사진을 업로드하고 다른 유저들과 공유할 수 있는 서비스

metadata를 저장할 database 설계

Key Generation Service Application

업로드하는 서버와 다운로드하는 서버 분리

- 읽기의 경우에는 캐시 덕분에 속도가 빠를 수 있음
- 따라서 쓰기를 하는 업로드 작업과 읽기를 하는 다운로드 작업 사이에는 스케일 차이가 존재
- 따라서 둘을 서로 다른 서버로 분리하여 스케일을 따로 관리

**Reliability**

- 서비스 어플리케이션 서버의 replica를 생성하여 reliability 확보

**Redundancy**

- 서비스 어플리케이션 서버의 fail over 관리
- 이미지 등 고객이 업로드한 파일을 잃어버리면 안되기 때문에 여러 스토리지에 복제하여 저장
- database 같은 경우에는 주기적으로 스냅샷을 유지하여 백업 관리
- 단일 장애 지점 없애기

### Reliability and Redundancy

Redundancy는 단일 장애 지점을 제거하고, 위기 상황에 필요한 경우 백업 및 스페어를 제공하는 역할

## Design: Dropbox

클라우드에 파일을 업로드 및 다운로드 할 수 있는 호스팅 서비스

Availability

Reliability and Durability

Scalability

여러 클라이언트와 서버의 동기화 필요

1. 큰 파일을 서로 주고 받으면 느리고 네트워크 낭비
    1. 따라서 파일을 chunk로 나누고, 변경분에 대해서만 업데이트되도록
    2. chunk 사이즈는 파일의 트래픽과 네트워크 대역폭를 고려하여 설정
2. 동기화 시 http polling 방식은 변경이 없어도 다시 요청하여 그로인해 서버가 busy함. 즉, scalable하지 않음
    1. 따라서 http long polling 방식을 사용하여 변경분이 있을때까지 서버는 request open
    2. 서버가 응답을 내려주고 요청을 완료하면, 클라이언트는 즉시 다시 요청을 보냄
3. 로컬에서 metadata를 위한 database 운영
    1. 서버와 연결이 끊길 때를 대비하여 클라이언트는 파일과 chunk, version 등을 내부적으로 관리
    2. 다시 온라인 상태가 되면, 클라이언트는 metadata를 서버와 동기화
    3. 데이터베이스는 sql와 nosql 둘 다 채택가능하며, nosql를 사용하는 경우에는 ACID를 보장하기 위해 논리적 프로그래밍 작성이 추가로 요구됨
4. metadata 동기화를 위한 synchronization service
    1. 이때도 전체 파일을 주고받지 말고, 두 버전 간의 차이(변경분, delta)만 전송하여 데이터 양을 줄임
    2. 서버와 클라이언트는 hash를 계산하여 chunk의 local copy를 업데이트할지 말지 결정
    3. 비슷한 hash 값을 가지는 경우에는 해당 chunk의 새로운 copy를 만들지 않고 기존 것 사용 가능
    4. 통신에 사용할 middle ware로는 global message queue를 채택할 수 있음

Message Queuing Service

- Request Queue
    - 모든 클라이언트가 공유하여 Synchronization Service가 차례대로 요청을 처리
- Response Queue
    - Queue에 담긴 메시지가 구독 후 삭제된다고 가정하면, 각 클라이언트 별로 queue를 생성하여 서버가 메시지 발송


File processing workflow

1. 클라이언트 A가 파일의 chunk를 클라우드 스토리지에 업로드
2. 클라이언트 A가 metadata를 업데이트하고 commit 처리
3. 클라이언트 A가 업데이트를 확인받으면, 변경 소식이 클라이언트 B와 클라이언트 C에게 전송
4. 클라이언트 B와 클라이언트 C가 metadata 변경되었다는 알람을 받으면 해당 chunk를 업데이트

Post-process deduplication

In-line deduplication

Metadata Partitioning

Vertical

## Design: Facebook Messenger

### workflow

1. 유저 A가 채팅 서버를 통해 유저 B에게 메시지를 보냄
2. 서버는 해당 메시지를 받고 유저 A에게 승인을 보냄
3. 서버는 해당 메시지를 데이터베이스에 저장하고 유저 B에게 보냄
4. 유저 B는 해당 메시지를 받고 서버에게 승인을 보냄
5. 서버는 유저 A에게 메시지가 유저 B에게 잘 전달됐음을 알림

### Message Handling

send/receive message 방법에는 다음 두 모델이 있음

1. pull model
    - 유저가 서버에게 주기적으로 메시지가 있는지 요청
    - 요청의 대부분은 empty이기 때문에 리소스 낭비
2. push model
    - 유저가 서버에 계속해서 연결하여 새로운 메시지가 있으면 응답을 받음
    - 많은 유저들이 서버와 커넥션을 맺고 있게됨

따라서 long polling 이나 websocket 전략 사용

Hbase 사용

key - value NoSQL으로, HDFS를 사용하고 작은 파일을 빠르게 적재할 수 있으며 row를 범위 단위로 스캔할 수 있다.

user id를 샤딩하여 key로 사용하면 각 노드 별로 분산되어 저장됨 


### Data Partitioning

대충 user ID를 샤딩 (hash modulo)해서 key로 저장하면 효율적이라는 뜻

message ID를 키로 사용하게되면 range scan에 불리하므로 하지마라

### Cache

15일 이내의 메시지는 자주 읽게 되므로 캐싱 해놓으면 이득

### Load balancing

클라이언트가 서버 요청할 때 트래픽 분산

서버가 캐시 서버로 요청할 때 트래픽 분산

### Fault tolerance and Replication

서버가 죽으면 유저는 다른 서버로 연결해야하는데, 다른 서버로 TCP 연결을 복구하는것이 매우 어렵다. 따라서 커넥션이 죽으면 클라이언트가 자동으로 새 커넥션을 연결해야 한다.

데이터를 잃어버리면 치명적이므로 데이터를 복구할 수 있도록 서로 다른 서버에 데이터를 복제하거나 Reed-Solomon encoding 기술을 사용해라

### Extended Requirements

그룹 챗 기능

푸시 알림 기능

## Design: Twitter

유저들이 트윗을 게시하고, 다른 사람이나 트윗을 팔로우할 수 있는 소셜 네트워크 시스템

### Requirements and Goals of the System

1. 유저는 새로운 트윗을 게시할 수 있다.
2. 유저는 다른 유저를 팔로우할 수 있다.
3. 유저는 트윗에 좋아요를 할 수 있다.
4. 서비스는 유저가 팔로우하는 사람들로부터 최상의 트윗이 포함된 타임라인을 생성하고 노출할 수 있어야한다.
5. 트윗은 사진과 비디오를 포함할 수 있다.

### Capacity Estimation and Constraints

## Design: Youtube or Netflix

### Requirements

1. 유저는 비디오를 업로드할 수 있다.
2. 유저는 비디오를 공유하고 볼 수 있다.
3. 유저는 제목으로 비디오를 검색할 수 있다.
4. 서비스는 비디오의 좋아요나 시청 수 같은 통계치를 기록할 수 있다.
5. 유저는 비디오에 댓글을 달거나 볼 수 있다.
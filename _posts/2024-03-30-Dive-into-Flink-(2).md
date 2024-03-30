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

이전 글에 이어 이번에는 Flink를 띄우고 테스트해보려고 한다. Kafka와 Flink를 Docker Container로 배포한 후 간단한 Job을 제출하여 스트리밍 애플리케이션이 잘 작동하는지 확인한다.

## Practice

1. Kafka 메시지의 발행과 구독은 콘솔을 이용한다.
2. Kafka topic 및 consumer 생성은 Kafka UI를 통해 확인한다.
3. Flink 애플리케이션은 Scala로 구현한다.

![image_01](/assets/img/posts/2024-03-30/image_01.png){: width="600" height="400" }

### Docker Setting

> 다음 Github 링크에 설정들을 상세하게 정리해 두었습니다.
> - [https://github.com/ivoryRabbit/play-data-with-docker/tree/master/flink](https://github.com/ivoryRabbit/play-data-with-docker/tree/master/flink){: target="_blank"}
{: .prompt-tip }

#### Kafka

실습이므로 카프카 브로커 개수는 1개, zookeeper를 별도로 띄우지 않기 위해 KRaft 지원이 가능한 3.4.1 버전 사용

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

CLI 커맨드 작성이 귀찮으므로 kafka UI 사용

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

Flink는 jobmanager와 taskmanager 역할을 하는 컨테이너를 각각 배포. jobmanager는 port를 뚫어준다

```yaml
flink-jobmanager:
  container_name: flink-jobmanager
  hostname: flink-jobmanager
  build:
      dockerfile: ./docker/flink/Dockerfile
  image: flink-dev
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
  build:
      dockerfile: ./docker/flink/Dockerfile
  image: flink-dev
  command: taskmanager
  environment:
    - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
  volumes:
    - ./docker/volume/flink/taskmanager:/data/flink
```

Scala 및 Kafka 관련 의존성을 이미지에 포함시켜주어야 함

```docker
FROM flink:1.18.1-scala_2.12-java11

ENV FLINK_VERSION=1.18.1
ENV FLINK_CONNECTOR_VERSION=1.17.2
ENV KAFKA_VERSION=3.4.1

RUN curl -L https://repo1.maven.org/maven2/org/apache/flink/flink-streaming-scala/${FLINK_VERSION}/flink-streaming-scala-${FLINK_VERSION}.jar \
    -o ${FLINK_HOME}/lib/flink-streaming-scala-${FLINK_VERSION}.jar
RUN curl -L https://repo1.maven.org/maven2/org/apache/flink/flink-connector-kafka/${FLINK_CONNECTOR_VERSION}/flink-connector-kafka-${FLINK_CONNECTOR_VERSION}.jar \
    -o ${FLINK_HOME}/lib/flink-connector-kafka-${FLINK_CONNECTOR_VERSION}.jar
RUN curl -L https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/${KAFKA_VERSION}/kafka-clients-${KAFKA_VERSION}.jar \
    -o ${FLINK_HOME}/lib/kafka-clients-${KAFKA_VERSION}.jar
RUN curl -L https://repo1.maven.org/maven2/org/apache/flink/flink-shaded-guava/30.1.1-jre-16.1/flink-shaded-guava-30.1.1-jre-16.1.jar \
    -o ${FLINK_HOME}/lib/flink-shaded-guava-30.1.1-jre-16.1.jar
```

확인

Kafka UI 스크린샷

Flink UI 스크린샷


토픽 생성

## Applications

```scala
object Job {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    val kafkaSource = KafkaSource.builder()
      .setBootstrapServers("kafka:9092")
      .setTopics("input.flink.dev")
      .setGroupId("flink-consumer-group")
      .setStartingOffsets(OffsetsInitializer.latest())
      .setValueOnlyDeserializer(new SimpleStringSchema())
      .setProperty("enable.auto.commit", "true")
      .setProperty("auto.commit.interval.ms", "1")
      .build()

    val kafkaSink = KafkaSink.builder()
      .setBootstrapServers("kafka:9092")
      .setRecordSerializer(
        KafkaRecordSerializationSchema.builder()
          .setTopic("output.flink.dev")
          .setValueSerializationSchema(new SimpleStringSchema())
          .build()
      )
      .setDeliveryGuarantee(DeliveryGuarantee.AT_LEAST_ONCE)
      .build()

    val streamLines = env.fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "Kafka Sink")
    streamLines.print()
    streamLines.sinkTo(kafkaSink)

    env.execute("Flink Streaming Scala Example")
  }
}
```

sbt를 이용해 컴파일

```bash
sbt clean assembly
```

jar 파일을 flink jobmanager로 업로드

```bash
curl -X POST http://localhost:8081/v1/jars/upload -H "Expect:" -F "jarfile=@./target/scala-2.12/flink-dev-assembly-0.1-SNAPSHOT.jar"
```

job 제출 스크린샷

consumer group이 등록되었는지 kafka ui로 확인
스크린샷

#### Kafka console producer

```bash
kafka-console-producer --broker-list localhost:9094 --topic input.flink.dev
```

#### Kafka console consumer

```bash
kafka-console-consumer --bootstrap-server localhost:9094 --topic output.flink.dev
```

메시지 발행 및 구독 스크린샷